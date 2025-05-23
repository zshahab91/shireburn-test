<script setup lang="ts">
import { ref, computed, watch } from 'vue'
import { useRouter } from 'vue-router'
import { exportToCSV, parseCSV, validateCSVFormat } from '@/utils/csvHelpers'

interface Employee {
  id: number
  code: string
  firstName: string
  lastName: string
  occupation: string
  department: string
  employmentDate: string
  terminationDate: string | null
}

// Add these interfaces after the Employee interface
interface Filter {
  field: keyof Employee | 'fullName'
  value: string
  operator: 'contains' | 'equals' | 'startsWith' | 'endsWith' | 'greaterThan' | 'lessThan'
}

// Props and emits
const props = defineProps<{
  employees: Employee[]
}>()

// Replace the existing defineEmits with this type-safe version
const emit = defineEmits<{
  (e: 'edit', employee: Employee): void
  (e: 'delete', employee: Employee): void
  (e: 'refresh'): void
  (e: 'loadMore'): void
}>()

// Router
const router = useRouter()

// State
const searchQuery = ref('')
const selectedDepartment = ref('')
const currentSort = ref('firstName')
const sortDirection = ref('asc')
const itemsPerPage = ref(10)
const currentPage = ref(1)
const pagesToShow = ref(5)
const fileInput = ref<HTMLInputElement | null>(null)
const importErrors = ref<string[]>([])
const showImportErrors = ref(false)

// Add these to the state section
const filters = ref<Filter[]>([])
const showFilterModal = ref(false)
const newFilter = ref<Filter>({
  field: 'fullName',
  value: '',
  operator: 'contains',
})

// Add a watch for itemsPerPage
watch(itemsPerPage, () => {
  // Reset to first page when items per page changes
  currentPage.value = 1
})

// Computed properties
const filteredEmployees = computed(() => {
  return props.employees.filter((employee) => {
    const fullName = `${employee.firstName} ${employee.lastName}`.toLowerCase()
    const matchesSearch =
      fullName.includes(searchQuery.value.toLowerCase()) ||
      employee.occupation.toLowerCase().includes(searchQuery.value.toLowerCase()) ||
      employee.department.toLowerCase().includes(searchQuery.value.toLowerCase())
    const matchesDepartment =
      !selectedDepartment.value || employee.department === selectedDepartment.value
    // Apply each filter
    const matchesMoreFiltered = filters.value.every((filter) => {
      let fieldValue: string | number | Date

      // Handle special case for full name
      if (filter.field === 'fullName') {
        fieldValue = `${employee.firstName} ${employee.lastName}`
      } else {
        fieldValue = employee[filter.field]
      }

      // Handle null values
      fieldValue = fieldValue ?? ''

      // Convert to lowercase for string comparisons
      const filterValue = filter.value.toLowerCase()
      fieldValue = String(fieldValue).toLowerCase()

      // Apply different operators
      switch (filter.operator) {
        case 'contains':
          return fieldValue.includes(filterValue)
        case 'equals':
          return fieldValue === filterValue
        case 'startsWith':
          return fieldValue.startsWith(filterValue)
        case 'endsWith':
          return fieldValue.endsWith(filterValue)
        case 'greaterThan':
          return new Date(fieldValue) > new Date(filterValue)
        case 'lessThan':
          return new Date(fieldValue) < new Date(filterValue)
        default:
          return true
      }
    })
    return matchesSearch && matchesDepartment && matchesMoreFiltered
  })
})

const sortedEmployees = computed(() => {
  return [...filteredEmployees.value].sort((a, b) => {
    let aValue = a[currentSort.value as keyof Employee]
    let bValue = b[currentSort.value as keyof Employee]

    if (currentSort.value === 'fullName') {
      aValue = `${a.firstName} ${a.lastName}`
      bValue = `${b.firstName} ${b.lastName}`
    }

    const direction = sortDirection.value === 'asc' ? 1 : -1
    return aValue < bValue ? -direction : direction
  })
})

const totalItems = computed(() => filteredEmployees.value.length)
const totalPages = computed(() => Math.ceil(totalItems.value / itemsPerPage.value))

// Removed unused 'paginatedEmployees' computed property

const pageNumbers = computed(() => {
  const pages = []
  const halfWay = Math.floor(pagesToShow.value / 2)
  let startPage = Math.max(currentPage.value - halfWay, 1)
  const endPage = Math.min(startPage + pagesToShow.value - 1, totalPages.value)

  if (endPage - startPage + 1 < pagesToShow.value) {
    startPage = Math.max(endPage - pagesToShow.value + 1, 1)
  }

  for (let i = startPage; i <= endPage; i++) {
    pages.push(i)
  }
  return pages
})

const displayedEmployees = computed(() => {
  const start = (currentPage.value - 1) * itemsPerPage.value
  const end = start + itemsPerPage.value
  return sortedEmployees.value.slice(start, end)
})

// Methods
const handleSort = (column: string) => {
  if (currentSort.value === column) {
    sortDirection.value = sortDirection.value === 'asc' ? 'desc' : 'asc'
  } else {
    currentSort.value = column
    sortDirection.value = 'asc'
  }
}

const changePage = (page: number) => {
  currentPage.value = Math.min(Math.max(1, page), totalPages.value)
}

const formatEmploymentStatus = (date: string) => {
  const employmentDate = new Date(date)
  const today = new Date()
  return employmentDate > today ? 'employed soon' : 'currently employed'
}

const formatTerminationStatus = (date: string | null) => {
  if (!date) return '-'
  const terminationDate = new Date(date)
  const today = new Date()
  return terminationDate > today ? 'to be terminated' : 'terminated'
}

const handleExport = () => {
  exportToCSV(props.employees, `employees-${new Date().toISOString().split('T')[0]}`)
}

const handleImportClick = () => {
  fileInput.value?.click()
}

const handleFileImport = async (event: Event) => {
  const target = event.target as HTMLInputElement
  if (!target.files?.length) return

  try {
    // Check file extension
    const file = target.files[0]
    if (!file.name.endsWith('.csv')) {
      importErrors.value = ['Please upload a CSV file']
      showImportErrors.value = true
      return
    }

    // Parse CSV
    const employees = await parseCSV(file)

    // Validate format
    const errors = validateCSVFormat(employees)
    if (errors.length > 0) {
      importErrors.value = errors
      showImportErrors.value = true
      return
    }

    // Upload employees with better error tracking
    const successfulUploads: string[] = []
    const failedUploads: string[] = []

    for (const employee of employees) {
      try {
        const response = await fetch('http://localhost:3000/employees', {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify(employee),
        })

        const data = await response.json()

        if (response.ok) {
          successfulUploads.push(`${employee.code}: ${employee.firstName} ${employee.lastName}`)
        } else {
          failedUploads.push(
            `${employee.code}: ${employee.firstName} ${employee.lastName} - ${data.message || 'Unknown error'}`,
          )
        }
      } catch {
        failedUploads.push(
          `${employee.code}: ${employee.firstName} ${employee.lastName} - Network error`,
        )
      }
    }

    // Show comprehensive import results
    if (failedUploads.length > 0) {
      importErrors.value = [
        `Successfully imported ${successfulUploads.length} employees`,
        'Failed to import the following employees:',
        ...failedUploads,
      ]
    } else {
      importErrors.value = [`Successfully imported all ${successfulUploads.length} employees`]
    }
    showImportErrors.value = true

    // Refresh grid only if we had successful imports
    if (successfulUploads.length > 0) {
      emit('refresh')
    }
  } catch (error) {
    console.error('Import failed:', error)
    importErrors.value = [
      'CSV Import Error:',
      (error as Error).message || 'Unknown error occurred',
      '',
      'Please ensure:',
      '• The CSV file has the correct headers',
      '• All required fields are filled',
      '• Dates are in YYYY-MM-DD format',
      '• Employee codes follow EMP### format',
    ]
    showImportErrors.value = true
  } finally {
    // Reset file input
    if (fileInput.value) {
      fileInput.value.value = ''
    }
  }
}

const handleDelete = (employee: Employee) => {
  emit('delete', employee)
}

// Add these methods
const addFilter = () => {
  filters.value.push({ ...newFilter.value })
  newFilter.value.value = ''
  showFilterModal.value = false
}

const removeFilter = (index: number) => {
  filters.value.splice(index, 1)
}
</script>

<template>
  <div class="space-y-4">
    <!-- Import/Export buttons -->
    <div class="flex justify-end space-x-4">
      <input ref="fileInput" type="file" accept=".csv" class="hidden" @change="handleFileImport" />
      <button
        @click="handleImportClick"
        class="px-4 py-2 text-sm font-medium text-gray-700 bg-white border border-gray-300 rounded-md shadow-sm hover:bg-gray-50 focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-purple-500"
      >
        Import CSV
      </button>
      <button
        @click="handleExport"
        class="px-4 py-2 text-sm font-medium text-gray-700 bg-white border border-gray-300 rounded-md shadow-sm hover:bg-gray-50 focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-purple-500"
      >
        Export CSV
      </button>
    </div>

    <!-- Import Results Modal -->
    <div v-if="showImportErrors" class="fixed inset-0 z-50 overflow-y-auto">
      <div
        class="flex items-end justify-center min-h-screen pt-4 px-4 pb-20 text-center sm:block sm:p-0"
      >
        <div
          class="fixed inset-0 bg-gray-500 bg-opacity-75 transition-opacity"
          @click="showImportErrors = false"
        ></div>

        <div
          class="inline-block align-bottom bg-white rounded-lg px-4 pt-5 pb-4 text-left overflow-hidden shadow-xl transform transition-all sm:my-8 sm:align-middle sm:max-w-lg sm:w-full sm:p-6"
        >
          <div>
            <div
              class="mx-auto flex items-center justify-center h-12 w-12 rounded-full"
              :class="[
                importErrors[0].includes('Successfully imported all')
                  ? 'bg-green-100'
                  : 'bg-yellow-100',
              ]"
            >
              <svg
                class="h-6 w-6"
                :class="[
                  importErrors[0].includes('Successfully imported all')
                    ? 'text-green-600'
                    : 'text-yellow-600',
                ]"
                fill="none"
                viewBox="0 0 24 24"
                stroke="currentColor"
              >
                <path
                  v-if="importErrors[0].includes('Successfully imported all')"
                  stroke-linecap="round"
                  stroke-linejoin="round"
                  stroke-width="2"
                  d="M5 13l4 4L19 7"
                />
                <path
                  v-else
                  stroke-linecap="round"
                  stroke-linejoin="round"
                  stroke-width="2"
                  d="M12 9v2m0 4h.01m-6.938 4h13.856c1.54 0 2.502-1.667 1.732-3L13.732 4c-.77-1.333-2.694-1.333-3.464 0L3.34 16c-.77 1.333.192 3 1.732 3z"
                />
              </svg>
            </div>
            <div class="mt-3 text-center sm:mt-5">
              <h3 class="text-lg leading-6 font-medium text-gray-900">Import Results</h3>
              <div class="mt-2 max-h-60 overflow-y-auto">
                <ul class="text-sm text-left space-y-2">
                  <li
                    v-for="(error, index) in importErrors"
                    :key="index"
                    :class="[error.includes('Successfully') ? 'text-green-600' : 'text-red-600']"
                  >
                    {{ error }}
                  </li>
                </ul>
              </div>
            </div>
          </div>
          <div class="mt-5 sm:mt-6">
            <button
              type="button"
              class="inline-flex justify-center w-full rounded-md border border-transparent shadow-sm px-4 py-2 bg-purple-600 text-base font-medium text-white hover:bg-purple-700 focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-purple-500 sm:text-sm"
              @click="showImportErrors = false"
            >
              Close
            </button>
          </div>
        </div>
      </div>
    </div>

    <!-- Search and Filter Section -->
    <div class="flex flex-col md:flex-row gap-4 mb-6">
      <div class="relative flex-grow">
        <input
          v-model="searchQuery"
          type="text"
          placeholder="Search employees..."
          class="w-full px-4 py-2 pr-10 border rounded-lg focus:ring-2 focus:ring-purple-500"
        />
        <button
          v-if="searchQuery"
          @click="searchQuery = ''"
          class="absolute right-2 top-1/2 -translate-y-1/2 text-gray-400 hover:text-gray-600"
        >
          <svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
            <path
              stroke-linecap="round"
              stroke-linejoin="round"
              stroke-width="2"
              d="M6 18L18 6M6 6l12 12"
            />
          </svg>
          <span class="sr-only">Clear search</span>
        </button>
      </div>
      <select
        v-model="selectedDepartment"
        class="px-4 py-2 border rounded-lg focus:ring-2 focus:ring-purple-500"
      >
        <option value="">All Departments</option>
        <option v-for="dept in [...new Set(employees.map((e) => e.department))]" :key="dept">
          {{ dept }}
        </option>
      </select>
    </div>

    <!-- Add this after the Search and Filter Section -->
    <div class="mb-4">
      <div class="flex items-center justify-between">
        <div class="flex flex-wrap gap-2">
          <div
            v-for="(filter, index) in filters"
            :key="index"
            class="flex items-center gap-2 px-3 py-1 bg-purple-50 border border-purple-200 rounded-lg"
          >
            <span class="text-sm text-purple-700">
              {{ filter.field }}: {{ filter.operator }} "{{ filter.value }}"
            </span>
            <button @click="removeFilter(index)" class="text-purple-400 hover:text-purple-600">
              <svg class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                <path
                  stroke-linecap="round"
                  stroke-linejoin="round"
                  stroke-width="2"
                  d="M6 18L18 6M6 6l12 12"
                />
              </svg>
            </button>
          </div>
        </div>
        <button
          @click="showFilterModal = true"
          class="px-3 py-1 text-sm text-purple-600 border border-purple-200 rounded-lg hover:bg-purple-50"
        >
          Add Filter
        </button>
      </div>
    </div>

    <!-- Add Filter Modal -->
    <div v-if="showFilterModal" class="fixed inset-0 z-50 overflow-y-auto">
      <div class="flex items-center justify-center min-h-screen p-4">
        <div class="fixed inset-0 bg-gray-500 bg-opacity-75" @click="showFilterModal = false"></div>
        <div class="relative bg-white rounded-lg p-6 max-w-md w-full">
          <h3 class="text-lg font-medium mb-4">Add Filter</h3>
          <div class="space-y-4">
            <div>
              <label class="block text-sm font-medium text-gray-700">Field</label>
              <select
                v-model="newFilter.field"
                class="mt-1 block w-full rounded-md border-gray-300 shadow-sm focus:border-purple-500 focus:ring-purple-500"
              >
                <option value="fullName">Full Name</option>
                <option value="code">Code</option>
                <option value="occupation">Occupation</option>
                <option value="department">Department</option>
                <option value="employmentDate">Employment Date</option>
                <option value="terminationDate">Termination Date</option>
              </select>
            </div>
            <div>
              <label class="block text-sm font-medium text-gray-700">Operator</label>
              <select
                v-model="newFilter.operator"
                class="mt-1 block w-full rounded-md border-gray-300 shadow-sm focus:border-purple-500 focus:ring-purple-500"
              >
                <option value="contains">Contains</option>
                <option value="equals">Equals</option>
                <option value="startsWith">Starts With</option>
                <option value="endsWith">Ends With</option>
                <option
                  v-if="['employmentDate', 'terminationDate'].includes(newFilter.field)"
                  value="greaterThan"
                >
                  After
                </option>
                <option
                  v-if="['employmentDate', 'terminationDate'].includes(newFilter.field)"
                  value="lessThan"
                >
                  Before
                </option>
              </select>
            </div>
            <div>
              <label class="block text-sm font-medium text-gray-700">Value</label>
              <input
                v-model="newFilter.value"
                :type="
                  ['employmentDate', 'terminationDate'].includes(newFilter.field) ? 'date' : 'text'
                "
                class="mt-1 block w-full rounded-md border-gray-300 shadow-sm focus:border-purple-500 focus:ring-purple-500"
              />
            </div>
          </div>
          <div class="mt-6 flex justify-end space-x-3">
            <button
              @click="showFilterModal = false"
              class="px-4 py-2 text-sm font-medium text-gray-700 bg-white border border-gray-300 rounded-md hover:bg-gray-50"
            >
              Cancel
            </button>
            <button
              @click="addFilter"
              :disabled="!newFilter.value"
              class="px-4 py-2 text-sm font-medium text-white bg-purple-600 rounded-md hover:bg-purple-700 disabled:opacity-50"
            >
              Add Filter
            </button>
          </div>
        </div>
      </div>
    </div>

    <!-- Employee Grid -->
    <div class="relative overflow-x-auto">
      <table class="min-w-full divide-y divide-gray-200">
        <thead class="bg-gray-50">
          <tr>
            <th
              v-for="header in [
                { key: 'code', label: 'Code' },
                { key: 'fullName', label: 'Employee Name' },
                { key: 'occupation', label: 'Occupation' },
                { key: 'department', label: 'Department' },
                { key: 'employmentDate', label: 'Employment Date' },
                { key: 'terminationDate', label: 'Termination Date' },
              ]"
               @click="handleSort(header.key)"
              :key="header.key"
              class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider"
            >
              {{ header.label }}
            </th>
            <th
              class="sticky right-0 bg-gray-50 px-6 py-3 text-center text-xs font-medium text-gray-500 uppercase tracking-wider shadow-l"
            >
              Actions
            </th>
          </tr>
        </thead>
        <tbody class="bg-white divide-y divide-gray-200">
          <tr v-for="employee in displayedEmployees" :key="employee.id">
            <td class="px-6 py-4 whitespace-nowrap">{{ employee.code }}</td>
            <td class="px-6 py-4 whitespace-nowrap">
              {{ employee.firstName }} {{ employee.lastName }}
            </td>
            <td class="px-6 py-4 whitespace-nowrap">{{ employee.occupation }}</td>
            <td class="px-6 py-4 whitespace-nowrap">{{ employee.department }}</td>
            <td class="px-6 py-4 whitespace-nowrap">
              {{ formatEmploymentStatus(employee.employmentDate) }}
            </td>
            <td class="px-6 py-4 whitespace-nowrap">
              {{ formatTerminationStatus(employee.terminationDate) }}
            </td>
            <td
              class="sticky right-0 bg-white px-6 py-4 whitespace-nowrap text-center text-sm font-medium space-x-2 shadow-l"
            >
              <button
                @click="router.push(`/employees/${employee.id}`)"
                class="text-purple-600 hover:text-purple-900"
                title="View Employee"
              >
                <svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                  <path
                    stroke-linecap="round"
                    stroke-linejoin="round"
                    stroke-width="2"
                    d="M15 12a3 3 0 11-6 0 3 3 0 016 0z"
                  />
                  <path
                    stroke-linecap="round"
                    stroke-linejoin="round"
                    stroke-width="2"
                    d="M2.458 12C3.732 7.943 7.523 5 12 5c4.478 0 8.268 2.943 9.542 7-1.274 4.057-5.064 7-9.542 7-4.477 0-8.268-2.943-9.542-7z"
                  />
                </svg>
              </button>
              <button
                @click="router.push(`/employees/${employee.id}/edit`)"
                class="text-blue-600 hover:text-blue-900"
                title="Edit Employee"
              >
                <svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                  <path
                    stroke-linecap="round"
                    stroke-linejoin="round"
                    stroke-width="2"
                    d="M11 5H6a2 2 0 00-2 2v11a2 2 0 002 2h11a2 2 0 002-2v-5m-1.414-9.414a2 2 0 112.828 2.828L11.828 15H9v-2.828l8.586-8.586z"
                  />
                </svg>
              </button>
              <button
                @click="handleDelete(employee)"
                class="text-red-600 hover:text-red-900"
                title="Delete Employee"
              >
                <svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                  <path
                    stroke-linecap="round"
                    stroke-linejoin="round"
                    stroke-width="2"
                    d="M19 7l-.867 12.142A2 2 0 0116.138 21H7.862a2 2 0 01-1.995-1.858L5 7m5 4v6m4-6v6m1-10V4a1 1 0 00-1-1h-4a1 1 0 00-1 1v3M4 7h16"
                  />
                </svg>
              </button>
            </td>
          </tr>
        </tbody>
      </table>
    </div>

    <!-- Pagination -->
    <div class="mt-4 flex flex-col sm:flex-row justify-between items-center gap-4">
      <!-- Items per page selector -->
      <div class="flex items-center gap-2">
        <label class="text-sm text-gray-600">Show</label>
        <select
          v-model="itemsPerPage"
          class="px-3 py-1 border rounded-lg focus:ring-2 focus:ring-purple-500"
        >
          <option :value="10">10</option>
          <option :value="25">25</option>
          <option :value="50">50</option>
          <option :value="100">100</option>
        </select>
        <span class="text-sm text-gray-600">entries</span>
      </div>

      <!-- Pagination info -->
      <div class="text-sm text-gray-600">
        Showing {{ (currentPage - 1) * itemsPerPage + 1 }} to
        {{ Math.min(currentPage * itemsPerPage, totalItems) }} of {{ totalItems }} entries
      </div>

      <!-- Pagination controls -->
      <div class="flex items-center gap-2">
        <button
          @click="changePage(1)"
          :disabled="currentPage === 1"
          class="px-3 py-1 border rounded-lg hover:bg-gray-50 disabled:opacity-50 disabled:cursor-not-allowed"
        >
          First
        </button>
        <button
          @click="changePage(currentPage - 1)"
          :disabled="currentPage === 1"
          class="px-3 py-1 border rounded-lg hover:bg-gray-50 disabled:opacity-50 disabled:cursor-not-allowed"
        >
          Previous
        </button>

        <button
          v-for="page in pageNumbers"
          :key="page"
          @click="changePage(page)"
          :class="[
            'px-3 py-1 border rounded-lg hover:bg-gray-50',
            currentPage === page ? 'bg-purple-100 text-purple-600 border-purple-200' : '',
          ]"
        >
          {{ page }}
        </button>

        <button
          @click="changePage(currentPage + 1)"
          :disabled="currentPage === totalPages"
          class="px-3 py-1 border rounded-lg hover:bg-gray-50 disabled:opacity-50 disabled:cursor-not-allowed"
        >
          Next
        </button>
        <button
          @click="changePage(totalPages)"
          :disabled="currentPage === totalPages"
          class="px-3 py-1 border rounded-lg hover:bg-gray-50 disabled:opacity-50 disabled:cursor-not-allowed"
        >
          Last
        </button>
      </div>
    </div>
  </div>
</template>

<style scoped>
.shadow-l {
  box-shadow: -2px 0 4px rgba(0, 0, 0, 0.05);
}

/* Ensure sticky positioning works properly */
.relative {
  position: relative;
}

.sticky {
  position: sticky;
  z-index: 1;
}
</style>
