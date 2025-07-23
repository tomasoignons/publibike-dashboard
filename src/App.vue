<template>
  <div class="min-h-screen bg-base-200">
    <div class="max-w-7xl mx-auto px-4 py-8">
      <!-- Header -->
      <div class="mb-8">
        <h1 class="text-4xl font-bold text-center mb-4">PubliBike Dashboard</h1>
        <p class="text-center text-base-content/70">Find and analyze PubliBike station availability</p>
      </div>

      <!-- Station Search -->
      <div class="card bg-base-100 shadow-xl mb-6">
        <div class="card-body">
          <h2 class="card-title mb-4">Search Station</h2>
          <div class="form-control relative">
            <input 
              type="text" 
              v-model="searchQuery" 
              @input="searchStations"
              placeholder="Type at least 3 letters to search for a station..." 
              class="input input-bordered w-full"
            />
            
            <!-- Search Results Dropdown -->
            <div v-if="searchResults.length > 0" class="absolute top-12 left-0 right-0 z-10">
              <div class="bg-base-100 border border-base-300 rounded-lg shadow-lg max-h-60 overflow-y-auto">
                <div 
                  v-for="station in searchResults" 
                  :key="station.id"
                  @click="selectStation(station)"
                  class="p-3 hover:bg-base-200 cursor-pointer border-b border-base-300 last:border-b-0"
                >
                  <div class="font-medium">{{ station.name }}</div>
                  <div class="text-sm text-base-content/70">{{ station.address }}, {{ station.city }}</div>
                </div>
              </div>
            </div>
          </div>
        </div>
      </div>

      <!-- Selected Station Info -->
      <div v-if="selectedStation" class="space-y-6">
        <!-- Station Details -->
        <div class="card bg-base-100 shadow-xl">
          <div class="card-body">
            <h2 class="card-title">{{ selectedStation.name }}</h2>
            <div class="grid grid-cols-1 md:grid-cols-2 gap-4 mt-4">
              <div>
                <p><span class="font-semibold">Address:</span> {{ selectedStation.address }}</p>
                <p><span class="font-semibold">City:</span> {{ selectedStation.city }} {{ selectedStation.zip }}</p>
                <p><span class="font-semibold">Status:</span> 
                  <span :class="selectedStation.is_active ? 'text-success' : 'text-error'">
                    {{ selectedStation.is_active ? 'Active' : 'Inactive' }}
                  </span>
                </p>
              </div>
              <div>
                <p><span class="font-semibold">Current Bikes:</span> {{ selectedStation.bike_count }}</p>
                <p><span class="font-semibold">Current E-Bikes:</span> {{ selectedStation.ebike_count }}</p>
                <p><span class="font-semibold">Total Capacity:</span> {{ selectedStation.total_vehicles }}</p>
              </div>
            </div>
            <div class="mt-4">
              <p class="text-sm text-base-content/70">Last updated: {{ formatDate(selectedStation.last_seen_at) }}</p>
            </div>
          </div>
        </div>

        <!-- Historical Chart - Bike Count Over Time -->
        <div class="card bg-base-100 shadow-xl">
          <div class="card-body">
            <h3 class="card-title mb-4">📊 Bike Count Over Time</h3>
            <!-- Time Period Selection -->
            <div class="flex flex-wrap gap-2 mb-4">
              <button 
                v-for="period in timePeriods" 
                :key="period.value"
                @click="selectedTimePeriod = period.value; loadBikeCountChart()"
                :class="['btn btn-sm', selectedTimePeriod === period.value ? 'btn-primary' : 'btn-outline']"
              >
                {{ period.label }}
              </button>
            </div>
            <!-- Chart Container -->
            <div class="w-full h-96">
              <canvas ref="bikeCountCanvas"></canvas>
            </div>
          </div>
        </div>

        <!-- Probability Chart - Odds of Getting a Bike -->
        <div class="card bg-base-100 shadow-xl">
          <div class="card-body">
            <h3 class="card-title mb-4">📈 Probability of Finding a Bike (by time of day)</h3>
            
            <!-- Day Filter Controls -->
            <div class="grid grid-cols-1 md:grid-cols-2 gap-4 mb-4">
              <div>
                <label class="label">
                  <span class="label-text font-semibold">Time Period:</span>
                </label>
                <div class="flex flex-wrap gap-2">
                  <button 
                    v-for="period in probabilityPeriods" 
                    :key="period.value"
                    @click="selectedProbabilityPeriod = period.value; loadProbabilityData()"
                    :class="['btn btn-sm', selectedProbabilityPeriod === period.value ? 'btn-primary' : 'btn-outline']"
                  >
                    {{ period.label }}
                  </button>
                </div>
              </div>
              
              <div>
                <label class="label">
                  <span class="label-text font-semibold">Day Filter:</span>
                </label>
                <div class="flex flex-wrap gap-2">
                  <button 
                    v-for="filter in dayFilters" 
                    :key="filter.value"
                    @click="selectedDayFilter = filter.value; loadProbabilityData()"
                    :class="['btn btn-sm', selectedDayFilter === filter.value ? 'btn-primary' : 'btn-outline']"
                  >
                    {{ filter.label }}
                  </button>
                </div>
              </div>
            </div>

            <!-- Current Filter Info -->
            <div class="alert alert-info mb-4">
              <span class="text-sm">
                Showing data for: {{ getCurrentFilterDescription() }}
                <br>
                <strong>Note:</strong> All times are displayed in local time (UTC{{ getUTCOffset() }})
              </span>
            </div>

            <!-- Probability Charts -->
            <div class="space-y-6">
              <!-- Regular Bikes Probability -->
              <div>
                <h4 class="font-semibold mb-2 text-primary">🚲 Regular Bikes Probability</h4>
                <p class="text-sm text-base-content/70 mb-3">Probability of finding a regular bike (5-minute intervals)</p>
                <div class="w-full h-64">
                  <canvas ref="bikeProbabilityCanvas"></canvas>
                </div>
              </div>

              <!-- E-Bikes Probability -->
              <div>
                <h4 class="font-semibold mb-2 text-secondary">⚡ E-Bikes Probability</h4>
                <p class="text-sm text-base-content/70 mb-3">Probability of finding an e-bike (5-minute intervals)</p>
                <div class="w-full h-64">
                  <canvas ref="ebikeProbabilityCanvas"></canvas>
                </div>
              </div>
            </div>
          </div>
        </div>
      </div>

      <!-- Loading indicator -->
      <div v-if="loading" class="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center z-50">
        <div class="bg-base-100 p-6 rounded-lg">
          <div class="loading loading-spinner loading-lg"></div>
          <p class="mt-2">Loading station data...</p>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, onMounted, watch } from 'vue'
import { createClient } from '@supabase/supabase-js'
import Chart from 'chart.js/auto'

// Initialize Supabase client
const supabaseUrl = import.meta.env.VITE_SUPABASE_URL
const supabaseKey = import.meta.env.VITE_SUPABASE_ANON_KEY
const supabase = createClient(supabaseUrl, supabaseKey, {db : { schema: 'publibike' }})

export default {
  name: 'PubliBikeDashboard',
  setup() {
    // Reactive data
    const searchQuery = ref('')
    const searchResults = ref([])
    const selectedStation = ref(null)
    const loading = ref(false)
    
    // Chart references
    const bikeCountCanvas = ref(null)
    const bikeProbabilityCanvas = ref(null)
    const ebikeProbabilityCanvas = ref(null)
    let bikeCountChart = null
    let bikeProbabilityChart = null
    let ebikeProbabilityChart = null

    // Time periods for bike count chart
    const selectedTimePeriod = ref('1day')
    const timePeriods = ref([
      { value: '1hour', label: 'Last Hour' },
      { value: '12hours', label: 'Last 12 Hours' },
      { value: '1day', label: 'Last Day' },
      { value: '3days', label: 'Last 3 Days' },
      { value: '1week', label: 'Last Week' }
    ])

    // Probability chart controls
    const selectedProbabilityPeriod = ref('1week')
    const selectedDayFilter = ref('all')
    const probabilityPeriods = ref([
      { value: '3days', label: 'Last 3 Days' },
      { value: '1week', label: 'Last Week' },
      { value: '2weeks', label: 'Last 2 Weeks' },
      { value: '1month', label: 'Last Month' }
    ])
    
    const dayFilters = ref([
      { value: 'all', label: 'All Days' },
      { value: 'same', label: 'Same Day of Week' },
      { value: 'weekdays', label: 'Weekdays Only' },
      { value: 'weekends', label: 'Weekends Only' }
    ])

    // Mock stations for fallback
    const mockStations = [
      { id: 1, name: 'Lausanne Gare', address: 'Place de la Gare 1', city: 'Lausanne', zip: '1003', is_active: true, bike_count: 5, ebike_count: 3, total_vehicles: 20, last_seen_at: '2025-07-20T10:30:00Z' },
      { id: 2, name: 'Lausanne Flon', address: 'Place de l\'Europe 1', city: 'Lausanne', zip: '1003', is_active: true, bike_count: 8, ebike_count: 2, total_vehicles: 25, last_seen_at: '2025-07-20T10:25:00Z' },
      { id: 3, name: 'Lausanne Riponne', address: 'Place Riponne', city: 'Lausanne', zip: '1005', is_active: true, bike_count: 12, ebike_count: 6, total_vehicles: 30, last_seen_at: '2025-07-20T10:20:00Z' },
      { id: 4, name: 'Lausanne Ouchy', address: 'Place du Port 1', city: 'Lausanne', zip: '1006', is_active: true, bike_count: 7, ebike_count: 4, total_vehicles: 22, last_seen_at: '2025-07-20T10:35:00Z' }
    ]

    // Search stations
    const searchStations = async () => {
      if (searchQuery.value.length < 3) {
        searchResults.value = []
        return
      }

      loading.value = true
      
      try {
        const { data, error } = await supabase
          .from('stations')
          .select('*')
          .ilike('name', `%${searchQuery.value}%`)
          .limit(10)
        
        if (error) {
          console.error('Supabase error:', error)
          searchResults.value = mockStations.filter(station => 
            station.name.toLowerCase().includes(searchQuery.value.toLowerCase())
          )
        } else {
          searchResults.value = data || []
        }
      } catch (error) {
        console.error('Error searching stations:', error)
        searchResults.value = mockStations.filter(station => 
          station.name.toLowerCase().includes(searchQuery.value.toLowerCase())
        )
      } finally {
        loading.value = false
      }
    }

    // Select station
    const selectStation = (station) => {
      selectedStation.value = station
      searchQuery.value = station.name
      searchResults.value = []
      loadStationData()
    }

    // Load all station data
    const loadStationData = async () => {
      if (!selectedStation.value) return
      loading.value = true
      
      try {
        await Promise.all([
          loadBikeCountChart(),
          loadProbabilityData()
        ])
      } catch (error) {
        console.error('Error loading station data:', error)
      } finally {
        loading.value = false
      }
    }

    // Load bike count chart data
    const loadBikeCountChart = async () => {
      if (!selectedStation.value || !bikeCountCanvas.value) return

      try {
        const chartData = await getBikeCountData(selectedStation.value.id, selectedTimePeriod.value)
        createBikeCountChart(chartData)
      } catch (error) {
        console.error('Error loading bike count chart:', error)
        createMockBikeCountChart()
      }
    }

    // Get bike count data from Supabase
    const getBikeCountData = async (stationId, period) => {
      let hours = 1
      switch (period) {
        case '1hour': hours = 1; break
        case '12hours': hours = 12; break
        case '1day': hours = 24; break
        case '3days': hours = 72; break
        case '1week': hours = 168; break
      }

      const startDate = new Date(Date.now() - hours * 60 * 60 * 1000)
      
      const { data, error } = await supabase
        .from('station_snapshots')
        .select('timestamp, bike_count, ebike_count')
        .eq('station_id', stationId)
        .gte('timestamp', startDate.toISOString())
        .order('timestamp', { ascending: true })

      if (error) throw error
      return data || []
    }

    // Create bike count chart
    const createBikeCountChart = (data) => {
      if (bikeCountChart) {
        bikeCountChart.destroy()
      }

      const ctx = bikeCountCanvas.value.getContext('2d')
      bikeCountChart = new Chart(ctx, {
        type: 'line',
        data: {
          labels: data.map(d => {
            const date = new Date(d.timestamp)
            return date.toLocaleTimeString('en-US', { 
              hour: '2-digit', 
              minute: '2-digit',
              ...(selectedTimePeriod.value.includes('day') || selectedTimePeriod.value === '1week' ? { 
                month: 'short', 
                day: 'numeric' 
              } : {})
            })
          }),
          datasets: [
            {
              label: 'Regular Bikes',
              data: data.map(d => d.bike_count),
              borderColor: '#3B82F6',
              backgroundColor: 'rgba(59, 130, 246, 0.1)',
              fill: true,
              tension: 0.4
            },
            {
              label: 'E-Bikes',
              data: data.map(d => d.ebike_count),
              borderColor: '#EF4444',
              backgroundColor: 'rgba(239, 68, 68, 0.1)',
              fill: true,
              tension: 0.4
            }
          ]
        },
        options: {
          responsive: true,
          maintainAspectRatio: false,
          scales: {
            y: {
              beginAtZero: true,
              title: {
                display: true,
                text: 'Number of Bikes'
              }
            }
          },
          plugins: {
            legend: {
              position: 'top',
            }
          }
        }
      })
    }

    // Create mock bike count chart
    const createMockBikeCountChart = () => {
      const now = new Date()
      const data = []
      let points = 24
      let interval = 60

      switch (selectedTimePeriod.value) {
        case '1hour': points = 12; interval = 5; break
        case '12hours': points = 144; interval = 5; break
        case '1day': points = 288; interval = 5; break
        case '3days': points = 216; interval = 20; break
        case '1week': points = 336; interval = 30; break
      }

      for (let i = points; i >= 0; i--) {
        const time = new Date(now.getTime() - (i * interval * 60 * 1000))
        const bike_count = Math.floor(Math.random() * 15) + 2
        const ebike_count = Math.floor(Math.random() * 10) + 1
        
        data.push({ timestamp: time.toISOString(), bike_count, ebike_count })
      }

      createBikeCountChart(data)
    }

    // Load probability data
    const loadProbabilityData = async () => {
      if (!selectedStation.value) return

      try {
        const probabilityData = await getProbabilityData(
          selectedStation.value.id, 
          selectedProbabilityPeriod.value,
          selectedDayFilter.value
        )
        
        createProbabilityCharts(probabilityData)
      } catch (error) {
        console.error('Error loading probability data:', error)
        createMockProbabilityCharts()
      }
    }

    // Get probability data from Supabase with day filtering
    const getProbabilityData = async (stationId, period, dayFilter) => {
      let days = 7
      switch (period) {
        case '3days': days = 3; break
        case '1week': days = 7; break
        case '2weeks': days = 14; break
        case '1month': days = 30; break
      }

      const startDate = new Date(Date.now() - days * 24 * 60 * 60 * 1000)
      
      const { data, error } = await supabase
        .from('station_snapshots')
        .select('timestamp, bike_count, ebike_count')
        .eq('station_id', stationId)
        .gte('timestamp', startDate.toISOString())
        .order('timestamp', { ascending: true })

      if (error) throw error
      
      // Filter data based on day filter
      const filteredData = filterDataByDay(data || [], dayFilter)
      
      // Calculate probabilities for 5-minute intervals
      return calculateProbabilitiesByTime(filteredData)
    }

    // Filter data based on day of week
    const filterDataByDay = (data, dayFilter) => {
      if (dayFilter === 'all') return data

      const now = new Date()
      const currentDayOfWeek = now.getDay() // 0 = Sunday, 1 = Monday, etc.

      return data.filter(snapshot => {
        const date = new Date(snapshot.timestamp)
        const dayOfWeek = date.getDay()

        switch (dayFilter) {
          case 'same':
            return dayOfWeek === currentDayOfWeek
          case 'weekdays':
            return dayOfWeek >= 1 && dayOfWeek <= 5 // Monday to Friday
          case 'weekends':
            return dayOfWeek === 0 || dayOfWeek === 6 // Sunday or Saturday
          default:
            return true
        }
      })
    }

    // Calculate probabilities by time of day (5-minute intervals)
    const calculateProbabilitiesByTime = (data) => {
      const timeSlots = {}
      
      // Initialize all 5-minute intervals
      for (let hour = 0; hour < 24; hour++) {
        for (let minute = 0; minute < 60; minute += 5) {
          const timeKey = `${hour.toString().padStart(2, '0')}:${minute.toString().padStart(2, '0')}`
          timeSlots[timeKey] = {
            bike: { total: 0, available: 0 },
            ebike: { total: 0, available: 0 }
          }
        }
      }

      // Process data
      data.forEach(snapshot => {
        const date = new Date(snapshot.timestamp)
        const hours = date.getHours()
        const minutes = date.getMinutes()
        const roundedMinutes = Math.floor(minutes / 5) * 5
        const timeKey = `${hours.toString().padStart(2, '0')}:${roundedMinutes.toString().padStart(2, '0')}`
        
        if (timeSlots[timeKey]) {
          // Regular bikes
          timeSlots[timeKey].bike.total++
          if (snapshot.bike_count > 0) {
            timeSlots[timeKey].bike.available++
          }
          
          // E-bikes
          timeSlots[timeKey].ebike.total++
          if (snapshot.ebike_count > 0) {
            timeSlots[timeKey].ebike.available++
          }
        }
      })

      // Calculate probabilities
      const result = {
        bike: [],
        ebike: []
      }

      Object.keys(timeSlots).sort().forEach(timeKey => {
        const slot = timeSlots[timeKey]
        
        // Regular bikes probability
        const bikeProbability = slot.bike.total > 0 
          ? Math.round((slot.bike.available / slot.bike.total) * 100)
          : 50 // Default to 50% if no data
          
        result.bike.push({
          time: timeKey,
          probability: bikeProbability,
          sampleSize: slot.bike.total
        })

        // E-bikes probability
        const ebikeProbability = slot.ebike.total > 0 
          ? Math.round((slot.ebike.available / slot.ebike.total) * 100)
          : 30 // Default to 30% for e-bikes if no data
          
        result.ebike.push({
          time: timeKey,
          probability: ebikeProbability,
          sampleSize: slot.ebike.total
        })
      })

      return result
    }

    // Create probability charts
    const createProbabilityCharts = (data) => {
      createSingleProbabilityChart(bikeProbabilityCanvas.value, data.bike, 'Regular Bikes', '#3B82F6', 'bikeProbabilityChart')
      createSingleProbabilityChart(ebikeProbabilityCanvas.value, data.ebike, 'E-Bikes', '#EF4444', 'ebikeProbabilityChart')
    }

    // Create a single probability chart
    const createSingleProbabilityChart = (canvas, data, label, color, chartType) => {
      if (!canvas) return

      // Destroy existing chart
      if (chartType === 'bikeProbabilityChart' && bikeProbabilityChart) {
        bikeProbabilityChart.destroy()
      } else if (chartType === 'ebikeProbabilityChart' && ebikeProbabilityChart) {
        ebikeProbabilityChart.destroy()
      }

      const ctx = canvas.getContext('2d')
      const chart = new Chart(ctx, {
        type: 'line',
        data: {
          labels: data.map(d => d.time),
          datasets: [{
            label: `${label} Availability Probability`,
            data: data.map(d => d.probability),
            borderColor: color,
            backgroundColor: color + '20',
            fill: true,
            tension: 0.4,
            pointRadius: 1,
            pointHoverRadius: 4
          }]
        },
        options: {
          responsive: true,
          maintainAspectRatio: false,
          scales: {
            y: {
              beginAtZero: true,
              max: 100,
              title: {
                display: true,
                text: 'Probability (%)'
              }
            },
            x: {
              title: {
                display: true,
                text: 'Time of Day'
              },
              ticks: {
                maxTicksLimit: 12,
                callback: function(value, index) {
                  return index % 12 === 0 ? this.getLabelForValue(value) : ''
                }
              }
            }
          },
          plugins: {
            legend: {
              display: false
            },
            tooltip: {
              callbacks: {
                afterLabel: function(context) {
                  const dataPoint = data[context.dataIndex]
                  return `Sample size: ${dataPoint.sampleSize} observations`
                }
              }
            }
          }
        }
      })

      // Store chart reference
      if (chartType === 'bikeProbabilityChart') {
        bikeProbabilityChart = chart
      } else {
        ebikeProbabilityChart = chart
      }
    }

    // Create mock probability charts
    const createMockProbabilityCharts = () => {
      const mockData = generateMockProbabilityData()
      createProbabilityCharts(mockData)
    }

    // Generate mock probability data
    const generateMockProbabilityData = () => {
      const bike = []
      const ebike = []

      for (let hour = 0; hour < 24; hour++) {
        for (let minute = 0; minute < 60; minute += 5) {
          const timeKey = `${hour.toString().padStart(2, '0')}:${minute.toString().padStart(2, '0')}`
          
          // Generate realistic probabilities based on time of day
          let bikeProbability = 70
          let ebikeProbability = 50
          
          // Lower availability during rush hours
          if ((hour >= 7 && hour <= 9) || (hour >= 17 && hour <= 19)) {
            bikeProbability = 40 + Math.random() * 30
            ebikeProbability = 20 + Math.random() * 30
          }
          // Higher availability late night/early morning
          else if (hour >= 23 || hour <= 6) {
            bikeProbability = 80 + Math.random() * 20
            ebikeProbability = 60 + Math.random() * 30
          }
          
          bike.push({
            time: timeKey,
            probability: Math.round(bikeProbability),
            sampleSize: Math.floor(Math.random() * 50) + 10
          })
          
          ebike.push({
            time: timeKey,
            probability: Math.round(ebikeProbability),
            sampleSize: Math.floor(Math.random() * 30) + 5
          })
        }
      }

      return { bike, ebike }
    }

    // Get current filter description
    const getCurrentFilterDescription = () => {
      const period = probabilityPeriods.value.find(p => p.value === selectedProbabilityPeriod.value)?.label || 'Unknown period'
      const filter = dayFilters.value.find(f => f.value === selectedDayFilter.value)?.label || 'Unknown filter'
      return `${period} - ${filter}`
    }

    // Get UTC offset for display
    const getUTCOffset = () => {
      const offset = new Date().getTimezoneOffset()
      const hours = Math.floor(Math.abs(offset) / 60)
      const minutes = Math.abs(offset) % 60
      const sign = offset <= 0 ? '+' : '-'
      return `${sign}${hours.toString().padStart(2, '0')}:${minutes.toString().padStart(2, '0')}`
    }

    // Utility function
    const formatDate = (dateString) => {
      return new Date(dateString).toLocaleString()
    }

    // Watchers
    watch(selectedTimePeriod, () => {
      if (selectedStation.value) {
        loadBikeCountChart()
      }
    })

    watch([selectedProbabilityPeriod, selectedDayFilter], () => {
      if (selectedStation.value) {
        loadProbabilityData()
      }
    })

    return {
      searchQuery,
      searchResults,
      selectedStation,
      loading,
      selectedTimePeriod,
      timePeriods,
      selectedProbabilityPeriod,
      selectedDayFilter,
      probabilityPeriods,
      dayFilters,
      bikeCountCanvas,
      bikeProbabilityCanvas,
      ebikeProbabilityCanvas,
      searchStations,
      selectStation,
      loadBikeCountChart,
      loadProbabilityData,
      getCurrentFilterDescription,
      getUTCOffset,
      formatDate
    }
  }
}
</script>