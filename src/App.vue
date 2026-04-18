<script setup lang="ts">
  import { ref, onMounted } from 'vue'
  import data from "./const/data.json"

  type GachaItem = {
    name: string
    url: string
    yearMonth: string
    week: number
    thumbnail: string
    kinds: number
  }

  // const gachadata = data as GachaItem[]
  const gachadata: GachaItem[] = data

  const showingYear = ref(
    Math.max(...gachadata.map(item => Number(item.yearMonth.split('-')[0])))
  )

  const isSearchOpen = ref(false)
  const search = ref("")
  const likedItems = ref<Record<string, boolean>>({})
  const showLikedOnly = ref(false)

  onMounted(() => {
    const saved = localStorage.getItem("likedItems")
    if (saved) {
      likedItems.value = JSON.parse(saved)
    }
  })

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

  

    // computed: {
    //   availableYears() {
    //       const map = {}

    //       this.gachadata.forEach(item => {
    //           const year = item.yearMonth.split("-")[0]

    //           if (!map[year]) {
    //               map[year] = 0
    //           }

    //           map[year]++
    //       })

    //       return Object.entries(map)
    //           .map(([year, count]) => ({
    //               year: Number(year),
    //               count
    //           }))
    //           .sort((a, b) => b.year - a.year)
    //   },


    //   groupedByYearMonth() {
    //       const grouped = {}

    //       let filtered = this.gachadata

    //       // ⭐ お気に入りモード
    //       if (this.showLikedOnly) {
    //           filtered = filtered.filter(item => this.likedItems[item.url])

    //           // 👉 年ごとにグループ
    //           filtered.forEach(item => {
    //               const year = item.yearMonth.split("-")[0]

    //               if (!grouped[year]) {
    //                   grouped[year] = []
    //               }

    //               grouped[year].push(item)
    //           })

    //          // 👉 年内で「月 → 週」で昇順ソート
    //           Object.keys(grouped).forEach(key => {
    //               grouped[key].sort((a, b) => {
    //                   const [, monthA] = (a.yearMonth || '').split('-')
    //                   const [, monthB] = (b.yearMonth || '').split('-')

    //                   // 月で比較
    //                   if (Number(monthA) !== Number(monthB)) {
    //                       return Number(monthA) - Number(monthB)
    //                   }

    //                   // 同じ月なら週で比較
    //                   return a.week - b.week
    //               })
    //           })

    //           // 👉 年で新しい順
    //           return Object.keys(grouped)
    //               .sort((a, b) => Number(b) - Number(a))
    //               .reduce((acc, key) => {
    //                   acc[key] = grouped[key]
    //                   return acc
    //               }, {})
    //       }

    //       // =========================
    //       // 通常モード（今まで通り）
    //       // =========================

    //       filtered = filtered.filter(item =>
    //           item.yearMonth.startsWith(String(this.showingYear))
    //       )

    //       filtered.forEach(item => {
    //           const yearMonth = item.yearMonth

    //           if (!grouped[yearMonth]) {
    //               grouped[yearMonth] = []
    //           }

    //           grouped[yearMonth].push(item)
    //       })

    //       Object.keys(grouped).forEach(key => {
    //           grouped[key].sort((a, b) => a.week - b.week)
    //       })

    //       return Object.keys(grouped)
    //           .sort((a, b) => {
    //               const [yearA, monthA] = a.split("-").map(Number)
    //               const [yearB, monthB] = b.split("-").map(Number)

    //               return yearB - yearA || monthB - monthA
    //           })
    //           .reduce((acc, key) => {
    //               acc[key] = grouped[key]
    //               return acc
    //           }, {})
    //   },

    //   filteredItems() {
    //     if (!this.search) return this.gachadata

    //     const keyword = this.search.toLowerCase()

    //     return this.gachadata.filter(item =>
    //       item.name.toLowerCase().includes(keyword)
    //     )
    //   },
    // },
   



  // }
</script>

<template>
  <div class="p-6">

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
        🔍 Search
      </button>

      <!-- ⭐ Liked toggle -->
      <button
        @click="showLikedOnly = !showLikedOnly"
        class="ml-2 px-4 py-2 rounded shadow transition"
        :class="showLikedOnly 
          ? 'bg-yellow-400 text-black' 
          : 'bg-gray-200 hover:bg-gray-300'"
      >
        ⭐ Liked
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


    <!-- ✅ カンバン -->
    <div class="flex gap-6 overflow-x-auto">
      <div
        v-for="(items, yearMonth) in groupedByYearMonth"
        :key="yearMonth"
        class="min-w-[300px]"
      >
        <h2 class="text-xl font-bold mb-4 p-2 bg-gray-200">{{ yearMonth }}（{{ items.length }}）</h2>
        <div
          v-for="item in items"
          :key="item.url"
          class="relative bg-white rounded mb-3 shadow border transition hover:shadow-md overflow-hidden"
          >
          <!-- <div style="display: block; height: 100px; widows: 100%; background-color: red;"></div> -->
          <a :href="item.url" target="_blank" rel="noopener noreferrer" class="block">
              <div class="block img-container relative h-[350px] w-full  overflow-hidden">
                <img :src="item.thumbnail" class="h-full mb-4 absolute top-0 left-[50%] transform -translate-x-1/2" style="max-width: unset;"/>
              </div>
            </a>

          <div class="p-3 ">

            <div class="flex justify-between items-center">
              <strong class="">{{ item.yearMonth.split('-')[1] }}月{{ item.week }}週 - {{ item.kinds }}種</strong>
              <div>
                <button
                  @click="toggleLike(item)"
                  class="text-xl transition"
                >
                  <span v-if="isLiked(item)" class="text-yellow-400">⭐</span>
                  <span v-else class="text-gray-300">☆</span>
                </button>
                <a
                  :href="'https://jp.mercari.com/search?keyword=' + item.name + ' めじるしアクセサリー'"
                  target="_blank"
                  rel="noopener noreferrer"
                  class="ml-4 inline-block px-2 py-1 bg-red-500 text-white text-sm font-semibold rounded-lg shadow hover:bg-red-600 hover:shadow-md transition"
                >
                  メルカリ
                </a>
  
              </div>
            </div>
            <p class="font-bold">{{ item.name }}</p>
          </div>

        </div>
      </div>
    </div>

  </div>
</template>