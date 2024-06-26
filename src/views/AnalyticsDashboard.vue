<template>
  <PageLayout>
    <div class="grid grid-cols-2 gap-10 p-8 text-center">
      <div class="items-center flex flex-col justify-end duration-150 hover:scale-105">
        <h1 class="text-tLightPurple text-6xl font-bold">Performance Analytics</h1>
        <p class="text-white">Keep track of how productive you have been through the weeks</p>
      </div>

      <div class="items-center flex flex-col justify-end duration-150 hover:scale-105">
        <h1 class="text-tLightPurple text-6xl font-bold flex items-center">
          Leaderboard
          <button
            class="ml-4 bg-tPurple text-white px-2 py-1 rounded-md text-sm hover:bg-tDarkPurple"
            @click="toggleFriendsLeaderboard"
          >
            {{ showFriendsLeaderboard ? 'Hide Friends Rank' : 'Show Friends Rank' }}
          </button>
        </h1>
        <p class="text-white">based on total productive hours on Twenify</p>
      </div>

      <!-- Analytics -->

      <div class="grid grid-cols-2 grid-rows-2 w-full gap-4">
        <div class="bg-white rounded-lg h-[30vh] flex flex-col p-4 duration-150 hover:scale-105">
          <p class="text-start font-semibold">Performance History</p>
          <div class="flex justify-center h-full items-center">
            <AnalyticsChart :data="subcollectionDateFocused"></AnalyticsChart>
          </div>
        </div>
        <div class="bg-white rounded-lg h-[30vh] flex flex-col p-4 duration-150 hover:scale-105">
          <p class="text-start font-semibold">Total Hours spent on twenify</p>
          <p class="text-[6rem] font-bold text-tYellow flex h-full justify-center items-center">
            {{ Math.round(totalHoursSpent) }}
          </p>
        </div>
        <div class="bg-white rounded-lg h-[30vh] flex flex-col p-4 duration-150 hover:scale-105">
          <p class="text-start font-semibold">Friends Ranking</p>
          <p
            class="text-[6rem] font-bold text-tLightPurple flex h-full justify-center items-center"
          >
            {{ formattedRanking(this.userFriendPosition) }}
          </p>
        </div>
        <div class="bg-white rounded-lg h-[30vh] flex flex-col p-4 duration-150 hover:scale-105">
          <p class="text-start font-semibold">Global Ranking</p>
          <p
            class="text-[6rem] font-bold text-tLightPurple flex h-full justify-center items-center"
          >
            {{ formattedRanking(this.userPosition) }}
          </p>
        </div>
      </div>

      <!-- Leaderboard -->

      <div class="flex items-start justify-center">
        <div v-if="leaderboardData" class="flex flex-col gap-2 w-[80%]">
          <div v-if="!showFriendsLeaderboard">
            <div
              v-for="(data, index) in leaderboardData"
              :key="index"
              :class="{
                'bg-tPurple text-tYellow p-3 w-full rounded-md duration-150 hover:scale-105 my-2':
                  data.Email === useremail,
                'bg-tPurple text-white p-3 w-full rounded-md duration-150 hover:scale-105 my-2':
                  data.Email !== useremail
              }"
            >
              <div class="flex px-3">
                <div class="flex w-[20%] text-start font-bold">{{ index + 1 }}</div>
                <div class="flex w-[60%] text-start font-bold">{{ data.Name }}</div>
                <div class="flex w-[20%] justify-end font-bold">
                  {{ Math.round(data.TotalHours) }} Hours
                </div>
              </div>
            </div>
          </div>
          <div v-else>
            <!-- Render friend details using friendLeaderboardData -->
            <div v-if="friendLeaderboardData.length === 0">
              <p class="text-white">You have no friends on Twenify yet!</p>
            </div>
            <div v-else>
              <div
                v-for="(friend, index) in friendLeaderboardData"
                :key="index"
                :class="{
                  'bg-tPurple text-tYellow p-3 w-full rounded-md duration-150 hover:scale-105 my-2':
                    friend.Email === useremail,
                  'bg-tPurple text-white p-3 w-full rounded-md duration-150 hover:scale-105 my-2':
                    friend.Email !== useremail
                }"
              >
                <div class="flex px-3">
                  <div class="flex w-[20%] text-start font-bold">{{ index + 1 }}</div>
                  <div class="flex w-[60%] text-start font-bold">{{ friend.Name }}</div>
                  <div class="flex w-[20%] justify-end font-bold">
                    {{ Math.round(friend.TotalHours) }} Hours
                  </div>
                </div>
              </div>
            </div>
          </div>
        </div>
        <div v-else>
          <p class="text-white">Loading leaderboard data...</p>
        </div>
      </div>
    </div>
  </PageLayout>
</template>

<script>
import firebaseApp from '../firebase.js'
import PageLayout from '@/components/PageLayout.vue'
import AnalyticsChart from '../components/AnalyticsChart.vue'
import { getAuth, onAuthStateChanged } from 'firebase/auth'
import {
  collection,
  getDocs,
  getFirestore,
  query,
  doc,
  getDoc,
  orderBy,
  limit,
  where
} from 'firebase/firestore'

const db = getFirestore(firebaseApp)

export default {
  name: 'AnalyticsDashboard',

  components: {
    PageLayout,
    AnalyticsChart
  },

  data() {
    return {
      user: false,
      useremail: null,
      userPosition: null,
      userFriendPosition: -1,
      totalHoursSpent: null,
      leaderboardData: null,
      friendLeaderboardData: [], // Initialize friendLeaderboardData as an empty array
      subcollectionDateFocused: null,
      showFriendsLeaderboard: false,
      friendsList: [] // Initialize friendsList as an empty array
    }
  },
  async mounted() {
    const auth = getAuth()
    onAuthStateChanged(auth, async (user) => {
      if (user) {
        this.user = user
        this.useremail = auth.currentUser.email
        await this.fetchLeaderboard()
        await this.fetchTotalHours()
        await this.fetchDateFocused(this.useremail)
      }
    })
  },

  methods: {
    formattedRanking(position) {
      if (position == -1) {
        return '-'
      } else {
        const rank = position + 1
        const suffix = ['st', 'nd', 'rd']
        const remainder = rank % 10
        return rank + (suffix[remainder - 1] || 'th')
      }
    },

    async fetchLeaderboard() {
      let querySnapshot
      const docRef = doc(db, 'Friends', this.useremail)
      const docSnap = await getDoc(docRef)
      const friendsEmails = docSnap.data().Friends // Retrieve emails of friends
      if (friendsEmails.length !== 0) {
        if (!friendsEmails.includes(this.useremail)) {
          friendsEmails.push(this.useremail)
        }

        // Fetch leaderboard data for friends
        querySnapshot = await getDocs(
          query(collection(db, 'Leaderboard'), where('Email', 'in', friendsEmails))
        )

        // Store fetched data in friendLeaderboardData
        this.friendLeaderboardData = querySnapshot.docs.map((doc) => doc.data())

        // Include user's own data in friendLeaderboardData
        //const userDoc = this.friendLeaderboardData.find((data) => data.Email === this.useremail);
        //if (userDoc) {
        // this.friendLeaderboardData.push(userDoc);
        //}

        // Sort friendLeaderboardData by TotalHours
        this.friendLeaderboardData.sort((a, b) => b.TotalHours - a.TotalHours)

        //console.log(this.friendLeaderboardData)

        // Retrieve and store user's position
        const userFriendPosition = this.friendLeaderboardData.findIndex(
          (data) => data.Email === this.useremail
        )
        this.userFriendPosition = userFriendPosition
      }
      console.log(this.userFriendPosition)

      // get the global ranking
      querySnapshot = await getDocs(
        query(collection(db, 'Leaderboard'), orderBy('TotalHours', 'desc'))
      )
      const allData = querySnapshot.docs.map((doc) => doc.data())
      this.userPosition = allData.findIndex((data) => data.Email === this.useremail)

      // get the top 10 leaderboard
      querySnapshot = await getDocs(
        query(collection(db, 'Leaderboard'), orderBy('TotalHours', 'desc'), limit(10))
      )
      this.leaderboardData = querySnapshot.docs.map((doc) => doc.data())
      console.log(this.userPosition)
    },

    async fetchTotalHours() {
      const docRef = doc(db, 'Total Hours', this.useremail)
      const docSnap = await getDoc(docRef)

      if (docSnap.exists()) {
        this.totalHoursSpent = docSnap.data().TotalHours
      }
    },

    async fetchDateFocused(useremail) {
      try {
        const userDocRef = doc(db, 'Users', useremail)
        const subcollectionRef = collection(userDocRef, 'DateFocused')
        const subcollectionQuery = query(subcollectionRef, orderBy('Date', 'desc'), limit(7))
        const subcollectionSnapshot = await getDocs(subcollectionQuery)

        let lastSevenDays = []
        for (let i = 6; i >= 0; i--) {
          const date = new Date()
          date.setDate(date.getDate() - i)
          const dateString = date.toISOString().split('T')[0] // Format as 'YYYY-MM-DD'
          lastSevenDays.push({
            Date: dateString,
            FocusedTime: 0 // Default value
          })
        }

        subcollectionSnapshot.forEach((doc) => {
          const data = doc.data()
          let dateStr = data.Date
          // Check if 'Date' is a Timestamp and convert it to a Date object
          if (data.Date.toDate) {
            dateStr = data.Date.toDate().toISOString().split('T')[0]
          }
          const dateIndex = lastSevenDays.findIndex((day) => day.Date === dateStr)
          if (dateIndex !== -1) {
            lastSevenDays[dateIndex].FocusedMinute = data.FocusedMinute
          }
        })

        this.subcollectionDateFocused = lastSevenDays
      } catch (error) {
        console.error('Error fetching user:', error)
      }
    },

    toggleFriendsLeaderboard() {
      this.showFriendsLeaderboard = !this.showFriendsLeaderboard
      if (!this.showFriendsLeaderboard) {
        // If switching back to user's leaderboard, fetch it again
        this.fetchLeaderboard()
      }
    }
  }
}
</script>
