<script setup lang="ts">
import type { Database } from '../types/supabase'
import { format } from 'date-fns'
import { onMounted, ref } from 'vue'
import { supabase } from '../lib/supabase'
import { useAuthStore } from '../stores/auth'

type Booking = Database['public']['Tables']['bookings']['Row']
type Profile = Database['public']['Tables']['profiles']['Row']
type Event = Database['public']['Tables']['events']['Row']
type Rental = Database['public']['Tables']['rentals']['Row']

const authStore = useAuthStore()
const bookings = ref<
  (Booking & { profiles: Profile, events: Event, rentals: Rental })[]
>([])
const loading = ref(true)
const viewMode = ref('guest') // 'guest' or 'host'
const statusFilter = ref('all') // 'pending', 'future', 'past', 'all'

async function fetchBookings() {
  try {
    bookings.value = []
    loading.value = true
    const now = new Date().toISOString()

    let query = supabase.from('bookings').select(`
      *,
      events (*),
      rentals (*),
      profiles!bookings_user_id_fkey (*)
    `)

    // Apply base filter based on view mode
    if (viewMode.value === 'guest') {
      if (!authStore.user?.id) {
        throw new Error('User not authenticated')
      }
      query = query.eq('user_id', authStore.user?.id)
    }
    else {
      // Host view: show bookings for events/rentals created by user or where user is a moderator
      const userId = authStore.user?.id
      if (!userId) {
        throw new Error('User not authenticated')
      }

      query = query.or(
        `events.creator_id.eq.${userId},rentals.creator_id.eq.${userId},events.moderators.cs.{${userId}},rentals.moderators.cs.{${userId}}`,
      )
    }

    // Apply status/time filter
    if (statusFilter.value === 'pending') {
      query = query.eq('status', 'pending')
    }
    else if (statusFilter.value === 'future') {
      query = query.gte('start_date', now).not('status', 'eq', 'rejected')
    }
    else if (statusFilter.value === 'past') {
      query = query.lt('start_date', now).not('status', 'eq', 'rejected')
    }

    query = query.order('start_date', { ascending: true })

    const { data, error } = await query
    if (error)
      throw error
    bookings.value = data as (Booking & {
      profiles: Profile
      events: Event
      rentals: Rental
    })[]
  }
  catch (error) {
    console.error('Error fetching bookings:', error)
  }
  finally {
    loading.value = false
  }
}

async function updateBookingStatus(
  bookingId: string,
  status: Database['public']['Enums']['booking_status'],
) {
  try {
    const { error } = await supabase
      .from('bookings')
      .update({ status })
      .eq('id', bookingId)

    if (error)
      throw error
    await fetchBookings()
  }
  catch (error) {
    console.error('Error updating booking:', error)
  }
}

onMounted(() => {
  fetchBookings()
})
</script>

<template>
  <div class="container px-4 py-8 mx-auto">
    <h1 class="mb-4 text-3xl md:text-4xl">
      Bookings
    </h1>

    <div class="flex flex-wrap items-center justify-end gap-3 mb-8">
      <!-- View Mode Toggle -->
      <div class="flex border-2 border-white">
        <button
          v-for="mode in ['guest', 'host']"
          :key="mode"
          class="px-3 py-2 text-sm md:px-4 md:text-base"
          :class="viewMode === mode ? 'bg-white text-black' : ''"
          @click="
            viewMode = mode;
            statusFilter = 'all';
            fetchBookings();
          "
        >
          {{ mode.charAt(0).toUpperCase() + mode.slice(1) }}
        </button>
      </div>

      <!-- Status Filter -->
      <select
        v-model="statusFilter"
        class="px-3 py-2 text-sm uppercase bg-black border-2 border-white md:text-base"
        @change="fetchBookings()"
      >
        <option value="all">
          All
        </option>
        <option value="pending">
          Pending
        </option>
        <option value="future">
          Future
        </option>
        <option value="past">
          Past
        </option>
      </select>
    </div>

    <div v-if="loading" class="flex justify-center py-12">
      <span class="loading loading-spinner loading-lg" />
    </div>

    <div v-else class="space-y-6">
      <div v-if="bookings.length === 0" class="py-8 text-center">
        <p class="text-white/60">
          No bookings found.
        </p>
      </div>

      <div v-else class="grid gap-6">
        <div
          v-for="booking in bookings"
          :key="booking.id"
          class="p-6 border-2 border-white"
        >
          <div class="flex items-start justify-between mb-4">
            <div>
              <h3 class="mb-2 text-xl">
                {{ booking.events?.title || booking.rentals?.title }}
              </h3>
              <p class="text-white/60">
                {{ format(new Date(booking.start_date), "PPP") }}
                {{
                  booking.end_date
                    ? ` - ${format(new Date(booking.end_date), "PPP")}`
                    : ""
                }}
              </p>
            </div>

            <span
              class="px-3 py-1 border"
              :class="{
                'bg-yellow-500 text-black': booking.status === 'pending',
                'bg-green-500 text-black': booking.status === 'approved',
                'bg-red-500 text-black': booking.status === 'rejected',
              }"
            >
              {{ booking.status }}
            </span>
          </div>

          <div v-if="booking.message" class="mb-4 prose-sm prose prose-invert">
            {{ booking.message }}
          </div>

          <!-- View Details Link -->
          <div class="flex justify-end mt-4">
            <router-link
              v-if="booking.events?.id"
              :to="`/app/events/${booking.events.id}`"
              class="px-6 py-2 btn-primary"
            >
              View Event Details
            </router-link>

            <router-link
              v-if="booking.rentals?.id"
              :to="`/app/rentals/${booking.rentals.id}`"
              class="px-6 py-2 btn-primary"
            >
              View Rental Details
            </router-link>
          </div>

          <!-- Management Actions -->
          <div
            v-if="viewMode === 'host' && booking.status === 'pending'"
            class="flex gap-4 mt-4"
          >
            <button
              class="px-4 py-2 btn-primary"
              @click="updateBookingStatus(booking.id, 'approved')"
            >
              Approve
            </button>

            <button
              class="px-4 py-2 btn-secondary"
              @click="updateBookingStatus(booking.id, 'rejected')"
            >
              Reject
            </button>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>
