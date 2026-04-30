<script setup lang="ts">
  import { ref, onMounted, computed, watchEffect } from 'vue'
  // import data from "./const/data.json"

  type GachaItem = {
    name: string
    url: string
    yearMonth: string
    week: number
    thumbnail: string
    kinds: number
  }

  // const gachadata = data as GachaItem[]
  // const gachadata: GachaItem[] = ref([]) as unknown as GachaItem[]
  const gachadata = ref<GachaItem[]>([])

  const showingYear = ref(0)

  watchEffect(() => {
    if (gachadata.value.length > 0) {
      showingYear.value = Math.max(
        ...gachadata.value.map(item =>
          Number(item.yearMonth.split('-')[0])
        )
      )
    }
  })

  const isSearchOpen = ref(false)
  const search = ref("")
  const likedItems = ref<Record<string, boolean>>({})
  const showLikedOnly = ref(false)

  declare global {
      interface Window {
          jsonpCallback?: (data: any) => void
      }
  }


  const baseUrl = "https://script.google.com/macros/s/AKfycbyI4gORtS87cI-CLop0ykUh57wIKnzhQg2baRcMFUbw7K5ad4kvJiqwewzTjWrrFRli/exec"

  const isLoading = ref(true)
  // const error = ref("")

  onMounted(() => {
    const saved = localStorage.getItem("likedItems")
    if (saved) {
      likedItems.value = JSON.parse(saved)
    }

    // get data from localStorage if exists
    const cachedData = localStorage.getItem('gachaData')
    if (cachedData) {
      gachadata.value = JSON.parse(cachedData)
      isLoading.value = false
    }



    fetchData()


  })

  const fetchData = () => {
    console.log("Fetching data...")
      const url = `${baseUrl}?callback=jsonpCallback&action=initApp`

      window.jsonpCallback = (data: any) => {
          console.log("API Response:", data)

          if (data.success) {
            const uniqueData: GachaItem[] = Array.from(
              new Map<string, GachaItem>(
                data.data.map((item: GachaItem) => [item.url, item])
              ).values()
            )

            const exisitingGachaData = localStorage.getItem('gachaData')

            const parsedExisting = exisitingGachaData
              ? JSON.parse(exisitingGachaData)
              : []

            if (parsedExisting.length !== uniqueData.length) {
              alert(
                "New data coming" +
                `\nPrevious count: ${parsedExisting.length}` +
                `\nNew count: ${uniqueData.length}`
              )

              isLoading.value = false
              gachadata.value = uniqueData
              localStorage.setItem('gachaData', JSON.stringify(uniqueData))
            }
          } else {
              console.error("Error:", data.message)
          }

          delete window.jsonpCallback
      }

      const script = document.createElement("script")
      script.src = url
      script.async = true

      document.body.appendChild(script)

      script.onload = () => {
          document.body.removeChild(script)
      }
  }

  const openSearch = () => {
    isSearchOpen.value = true
  }

  const closeSearch = () => {
    isSearchOpen.value = false
    search.value = "" // 検索キーワードもリセット
  }

  const toggleLike = (item: GachaItem) => {
    if (likedItems.value[item.url]) {
      // 解除
      delete likedItems.value[item.url]        // Vue3でもOK
    } else {
      // 追加
      likedItems.value[item.url] = true
    }

    // 💾 保存
    localStorage.setItem("likedItems", JSON.stringify(likedItems.value))
  }

  const isLiked = (item: GachaItem) => {
    return !!likedItems.value[item.url]
  }

  const availableYears = computed(() => {
      const map: Record<string, number> = {}

      gachadata.value.forEach(item => {
          const year = item.yearMonth.split("-")[0]

          if (!map[year]) {
              map[year] = 0
          }

          map[year]++
      })

      return Object.entries(map)
          .map(([year, count]) => ({
              year: Number(year),
              count
          }))
          .sort((a, b) => b.year - a.year)
  })

  const filteredItems = computed<GachaItem[]>(() => {
      if (!search.value) return [...gachadata.value]

      const keyword = search.value.toLowerCase()

      return gachadata.value.filter(item =>
          item.name.toLowerCase().includes(keyword)
      )
  })
  const groupedByYearMonth = computed(() => {
      const grouped: Record<string, GachaItem[]> = {}

      let filtered = gachadata.value

      // ⭐ liked mode
      if (showLikedOnly.value) {
          filtered = filtered.filter(item => likedItems.value[item.url])

          filtered.forEach(item => {
              const year = item.yearMonth.split("-")[0]

              if (!grouped[year]) {
                  grouped[year] = []
              }

              grouped[year].push(item)
          })

          Object.keys(grouped).forEach(key => {
              grouped[key].sort((a, b) => {
                  const [, monthA] = a.yearMonth.split('-')
                  const [, monthB] = b.yearMonth.split('-')

                  if (Number(monthA) !== Number(monthB)) {
                      return Number(monthA) - Number(monthB)
                  }

                  return a.week - b.week
              })
          })

          return Object.keys(grouped)
              .sort((a, b) => Number(b) - Number(a))
              .reduce((acc: Record<string, GachaItem[]>, key) => {
                  acc[key] = grouped[key]
                  return acc
              }, {})
      }

      // normal mode
      filtered = filtered.filter(item =>
          item.yearMonth.startsWith(String(showingYear.value))
      )

      filtered.forEach(item => {
          const yearMonth = item.yearMonth

          if (!grouped[yearMonth]) {
              grouped[yearMonth] = []
          }

          grouped[yearMonth].push(item)
      })

      Object.keys(grouped).forEach(key => {
          grouped[key].sort((a, b) => {
              // ① week (descending)
              if (b.week !== a.week) {
                  return b.week - a.week
              }

              // ② name (ascending)
              return a.name.localeCompare(b.name, 'ja')
          })
      })

      return Object.keys(grouped)
          .sort((a, b) => {
              const [yearA, monthA] = a.split("-").map(Number)
              const [yearB, monthB] = b.split("-").map(Number)

              return yearB - yearA || monthB - monthA
          })
          .reduce((acc: Record<string, GachaItem[]>, key) => {
              acc[key] = grouped[key]
              return acc
          }, {})
  })

  const isBandai = (url: string) => {
    return url.includes("gashapon.jp")
  }

</script>

<template>
  <div class="p-3 pt-1">

    <!-- ✅ 年選択 -->
    <div class="mb-6">
      <select v-if="!showLikedOnly" v-model="showingYear" class="border p-2 rounded">
        <option
          v-for="item in availableYears"
          :key="item.year"
          :value="item.year"
        >
          {{ item.year }}年 ({{ item.count }})
        </option>
      </select>


      <!-- 🔍 open button -->
      <button
        @click="openSearch"
        class="ml-4 px-4 py-2 bg-indigo-600 text-white rounded shadow hover:bg-indigo-700"
      >
        🔍
      </button>

      <!-- ⭐ Liked toggle -->
      <button
        @click="showLikedOnly = !showLikedOnly"
        class="ml-2 px-4 py-2 rounded shadow transition"
        :class="showLikedOnly 
          ? 'bg-yellow-400 text-black' 
          : 'bg-gray-200 hover:bg-gray-300'"
      >
        ⭐
      </button>

      <!-- 🪟 modal -->
      <div
        v-if="isSearchOpen"
        class="fixed inset-0 bg-black/50 flex items-center justify-center z-50"
        >
        <div class="bg-white w-[500px] max-h-[80vh] rounded-xl p-4 shadow-lg overflow-hidden">

          <!-- header -->
          <div class="flex justify-between items-center mb-3">
            <h2 class="font-bold text-lg">Search</h2>
            <button @click="closeSearch">✖</button>
          </div>

          <!-- input -->
          <input
            v-model="search"
            placeholder="検索..."
            class="w-full border p-2 rounded mb-3"
          />

          <!-- results -->
          <div class="overflow-y-auto max-h-[60vh]">
            <div
              v-for="item in filteredItems"
              :key="item.url"
              class="flex gap-3 items-center p-2 hover:bg-gray-100 rounded cursor-pointer"
            >
              <img v-if="filteredItems.length <= 25" :src="item.thumbnail" class="w-12 h-12 object-cover rounded" />

              <div class="flex-1">
                <p class="font-semibold">{{ item.name }}</p>
                <p class="text-xs text-gray-500">
                  {{ item.yearMonth }} / Week {{ item.week }}
                </p>
              </div>

              <a
                :href="item.url"
                target="_blank"
                class="text-blue-500 text-sm"
              >
                詳細
              </a>
            </div>
          </div>

        </div>
      </div>

    </div>

    <div v-if="isLoading">
      <p class="text-center text-gray-500">Loading...</p>
    </div>



    <!-- ✅ カンバン -->
    <div class="flex gap-3 overflow-x-auto">
      <div
        v-for="(items, yearMonth) in groupedByYearMonth"
        :key="yearMonth"
      >
        <h2 class="text-xl font-bold mb-4 p-1 bg-gray-500 w-[185px]">{{ yearMonth }}（{{ items.length }}）</h2>
        <div class="h-[665px] md:h-auto overflow-y-auto">
          <div
            v-for="item in items"
            :key="item.url"
            class="relative rounded mb-3 shadow transition w-full hover:shadow-md overflow-hidden"
            :class="isBandai(item.url) ? 'bg-white' : 'bg-red-100'"
            >
            <!-- <div style="display: block; height: 100px; widows: 100%; background-color: red;"></div> -->
            <a :href="item.url" target="_blank" rel="noopener noreferrer" class="block">
                <div 
                  class="block img-container relative w-full  overflow-hidden"
                  :class="isBandai(item.url) ? 'h-[215px]' : 'h-[185px]'"
                  >
                  <img :src="item.thumbnail" class="h-full mb-4 absolute top-0 left-[50%] transform -translate-x-1/2" style="max-width: unset;"/>
                </div>
              </a>
  
            <div class="p-2">
  
              <div class="flex justify-between items-center text-xs text-gray-400">
                <strong v-if="isBandai(item.url)" class="">{{ item.yearMonth.split('-')[1] }}月{{ item.week }}週 - {{ item.kinds }}種</strong>
                <strong v-else class="">{{ item.yearMonth.split('-')[1] }}月 - {{ item.kinds }}種 - <span class="text-red-500">宝</span></strong>
                <div class="flex justify-between items-center gap-2">
                  <button
                    @click="toggleLike(item)"
                    class="text-xl transition"
                  >
                    <span v-if="isLiked(item)" class="text-yellow-400">⭐</span>
                    <span v-else class="">☆</span>
                  </button>
                  <a
                    :href="'https://jp.mercari.com/search?keyword=' + item.name + ' めじるしアクセサリー'"
                    target="_blank"
                    rel="noopener noreferrer"
                    class="inline-block border border-black rounded shadow"
                  >
                    <img src="./assets/new_mercari_icon_iOS.png" alt="home icon" class="w-[20px] h-auto"/>
                  </a>
    
                </div>
              </div>
              <p class="font-bold text-sm text-black text-left">{{ item.name }}</p>
            </div>
  
          </div>
        </div>
      </div>
    </div>

  </div>
</template>