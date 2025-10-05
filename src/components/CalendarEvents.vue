<template>
  <div class="text-white">
  <h1 class="text-4xl font-bold mb-4 text-center">Events for {{ displayDate }}</h1>
    <div v-if="loading" class="text-center py-8 text-2xl">Loading events...</div>
    <div v-else-if="events.length === 0" class="text-center py-8 text-2xl">No events left for today.</div>
    <ul v-else class="space-y-4">
  <li v-for="event in events" :key="event.uid" class=" text-2xl bg-white bg-opacity-10 rounded p-4 flex flex-col">
        <span class="font-semibold">{{ event.displayTime }}</span>
        <span class="text-2xl">{{ event.summary }}</span>
  <span v-if="event.location && event.location !== 'The Well, a United Methodist Church'" class="text-sm text-blue-200 mt-1">{{ event.location }}</span>
      </li>
    </ul>
  </div>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount } from 'vue'
import { DateTime } from 'luxon'
import * as ICAL from 'ical.js'
const ICALCore = ICAL.default || ICAL;

const events = ref([])
const loading = ref(true)
const displayDate = ref(null)
let timer = null

async function fetchEvents() {
  console.log('[fetchEvents] Starting fetch');
  loading.value = true;
  try {
    const icalUrl = 'https://thewellmn.ccbchurch.com/w_calendar_sub.ics';
    console.log('[fetchEvents] Fetching iCal:', icalUrl);
    const resp = await fetch(`https://corsproxy.io/?${encodeURIComponent(icalUrl)}`);
    console.log('[fetchEvents] Fetch response:', resp);
    const icalText = await resp.text();
    console.log('[fetchEvents] iCal text length:', icalText.length);
    const jcalData = (ICAL.parse || ICAL.default?.parse)(icalText);
    console.log('[fetchEvents] Parsed jCal data:', jcalData);
    const comp = new ICALCore.Component(jcalData);
    const vevents = comp.getAllSubcomponents('vevent');
    console.log('[fetchEvents] Found vevents:', vevents.length);
    const now = DateTime.now().setZone('America/Chicago');
    let day = now;
    let found = false;
    let result = [];
    let attempts = 0;
    while (!found && attempts < 7) { // look up to 7 days ahead
      const dayStart = day.startOf('day');
      const dayEnd = day.endOf('day');
      result = [];
      for (const v of vevents) {
        const event = new ICALCore.Event(v);
        if (event.isRecurring()) {
          const expand = new ICALCore.RecurExpansion({component: v, dtstart: event.startDate});
          let recurCount = 0;
          const maxRecurrences = 1000;
          while (expand.next()) {
            recurCount++;
            if (recurCount > maxRecurrences) {
              console.warn('[fetchEvents] Recurrence expansion exceeded max iterations, breaking to avoid infinite loop.');
              break;
            }
            const occ = expand.last;
            const occStart = DateTime.fromJSDate(occ.toJSDate(), {zone: 'America/Chicago'});
            const occEnd = occStart.plus({seconds: event.duration.toSeconds()});
            if (occStart < dayStart || occStart > dayEnd) continue;
            if (occEnd < now && day.hasSame(now, 'day')) continue;
            result.push(formatEvent(event, occStart, occEnd));
            if (result.length >= 7) break;
          }
        } else {
          const start = DateTime.fromJSDate(event.startDate.toJSDate(), {zone: 'America/Chicago'});
          const end = DateTime.fromJSDate(event.endDate.toJSDate(), {zone: 'America/Chicago'});
          if (start < dayStart || start > dayEnd) continue;
          if (end < now && day.hasSame(now, 'day')) continue;
          result.push(formatEvent(event, start, end));
        }
        if (result.length >= 7) break;
      }
      if (result.length > 0) {
        found = true;
        break;
      }
      day = day.plus({ days: 1 });
      attempts++;
    }
    events.value = result.sort((a, b) => a.start - b.start).slice(0, 7);
    displayDate.value = day.toFormat('cccc, LLLL d, yyyy');
    document.title = `Events for ${displayDate.value}`;
    console.log('[fetchEvents] Final events:', events.value, 'for', displayDate.value);
  } catch (e) {
    console.error('[fetchEvents] Error:', e);
    events.value = [];
  }
  loading.value = false;
}

function formatEvent(event, start, end) {
  const isAllDay = event.startDate.isDate
  let displayTime = ''
  if (isAllDay) {
    displayTime = 'All Day'
  } else {
    const startStr = start.toFormat('h:mm')
    const endStr = end.toFormat('h:mm')
    const startPeriod = start.toFormat('a')
    const endPeriod = end.toFormat('a')
    if (startPeriod === endPeriod) {
      displayTime = `${startStr}-${endStr} ${startPeriod}`
    } else {
      displayTime = `${startStr} ${startPeriod}-${endStr} ${endPeriod}`
    }
  }
  return {
    uid: event.uid + start.toISO(),
    summary: event.summary,
    displayTime,
    location: event.location || '',
    start,
    end,
  }
}

function setupRefresh() {
  if (timer) clearInterval(timer)
  timer = setInterval(() => fetchEvents(), 15 * 60 * 1000)
}

onMounted(() => {
  fetchEvents()
  setupRefresh()
})

onBeforeUnmount(() => {
  if (timer) clearInterval(timer)
})
</script>

<style scoped>

</style>
