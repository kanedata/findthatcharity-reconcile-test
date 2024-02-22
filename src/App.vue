<script setup></script>

<script>
const defaultTestOrganisations = [
  ['Alzheimer Society', 'GB-CHC-296645'],
  ['Mind', 'GB-CHC-219830'],
  ['Voluntary Sector Studies Network', 'GB-CHC-1197827'],
  ['Friends of Lime Tree Primary Surbiton', 'GB-CHC-1186755'],
  ['James Bond Fan Club CIC', 'GB-COH-14545528'],
  ['James Bond Fan Club Community Interest Company', 'GB-COH-14545528'],
  ['James Bond Fan Club Community Interest Company', 'GB-COH-14545528'],
  ['Big Lottery Fund', 'GB-GOR-PB188'],
  ['National Lottery Community Fund', 'GB-GOR-PB188'],
  ['Royal Borough of Kingston Upon Thames', 'GB-LAE-KTT'],
  ['Kingston Upon Thames Council', 'GB-LAE-KTT']
]
export default {
  data() {
    return {
      version: 'live',
      testOrganisations: [...defaultTestOrganisations],
      results: [],
      loading: false,
      addOrganisationName: null,
      addOrganisationId: null
    }
  },
  mounted() {
    const storedSettings = localStorage.getItem('ftcReconcileStoredSettings')
    if (storedSettings) {
      var settings = JSON.parse(storedSettings)
      if (settings.testOrganisations) {
        this.testOrganisations = settings.testOrganisations
      }
      if (settings.version) {
        this.version = settings.version
      }
    }
  },
  watch: {
    testOrganisations: function () {
      this.storeSettings()
    },
    version: function () {
      this.storeSettings()
    }
  },
  computed: {
    endpoint() {
      return this.version === 'live'
        ? 'https://findthatcharity.uk/api/v1/reconcile/'
        : 'http://localhost:8000/api/v1/reconcile/'
    },
    queries() {
      return Object.fromEntries(
        this.testOrganisations.map(([query], index) => [`q${index}`, { query: query }])
      )
    },
    testResults() {
      return this.testOrganisations.map(([query, expectedId], index) => {
        const result = this.results[`q${index}`]?.result
        if (result && result.length > 0) {
          var match_ids = result.map((match) => match.id)
          return {
            query,
            expectedId,
            topMatchName: result[0].name,
            topMatchId: match_ids[0],
            correctMatch: match_ids[0] === expectedId,
            inPotentialMatches: match_ids.includes(expectedId)
          }
        } else {
          return {
            query,
            expectedId,
            topMatchName: null,
            topMatchId: null,
            correctMatch: null,
            inPotentialMatches: null
          }
        }
      })
    }
  },
  methods: {
    runTests() {
      this.loading = true
      const data = new URLSearchParams()
      data.append('queries', JSON.stringify(this.queries))
      fetch(this.endpoint, {
        method: 'POST',
        body: data
      })
        .then((response) => response.json())
        .then((data) => {
          this.results = data
          this.loading = false
        })
    },
    updateOrganisationName(index, value) {
      this.testOrganisations[index][0] = value
    },
    updateOrganisationId(index, value) {
      this.testOrganisations[index][1] = value
    },
    addOrganisation() {
      this.testOrganisations.push([this.addOrganisationName, this.addOrganisationId])
      this.addOrganisationName = null
      this.addOrganisationId = null
    },
    storeSettings() {
      localStorage.setItem(
        'ftcReconcileStoredSettings',
        JSON.stringify({
          version: this.version,
          testOrganisations: this.testOrganisations
        })
      )
    },
    clearSettings() {
      localStorage.removeItem('ftcReconcileStoredSettings')
      this.testOrganisations = [...defaultTestOrganisations]
      this.version = 'live'
    }
  }
}
</script>

<template>
  <header class="primary-container">
    <nav>
      <h5 class="max center-align">Find that Charity Reconciliation Test</h5>
    </nav>
  </header>

  <main class="responsive">
    <div class="medium-space"></div>
    <nav>
      <div class="field suffix border">
        <select v-model="version">
          <option value="live">Use live version</option>
          <option value="dev">Use development version</option>
        </select>
        <i>arrow_drop_down</i>
      </div>
      <button @click="runTests" class="primary">Run tests</button>
      <button @click="clearSettings" class="fill">Clear settings</button>
      <span v-if="loading">Loading...</span>
    </nav>
    <div class="medium-space"></div>
    <table class="table result-table">
      <thead>
        <tr>
          <th class="min"></th>
          <th class="">Query</th>
          <th class="">Expected ID</th>
          <th class="">Top match name</th>
          <th class="min">Top match ID</th>
          <th class="min">Correct match</th>
          <th class="min">In matches</th>
        </tr>
      </thead>
      <tbody>
        <tr
          v-for="(organisation, index) in testResults"
          :key="index"
          :class="{
            'bg-washed-green': organisation.correctMatch,
            'bg-washed-red': !organisation.correctMatch && !organisation.inPotentialMatches,
            'bg-light-yellow': !organisation.correctMatch && organisation.inPotentialMatches
          }"
        >
          <td class="min">
            <button
              @click="testOrganisations.splice(index, 1)"
              class="small surface-container-highest circle left-round top-round"
            >
              <i>remove</i>
            </button>
          </td>
          <td class="min">
            <div class="field small">
              <input
                type="text"
                :class="{
                  'bg-washed-green': organisation.correctMatch,
                  'bg-washed-red': !organisation.correctMatch && !organisation.inPotentialMatches,
                  'bg-light-yellow': !organisation.correctMatch && organisation.inPotentialMatches
                }"
                class="org-name"
                :value="organisation.query"
                @input="(event) => updateOrganisationName(index, event.target.value)"
              />
            </div>
          </td>
          <td class="min">
            <div class="field small">
              <input
                :class="{
                  'bg-washed-green': organisation.correctMatch,
                  'bg-washed-red': !organisation.correctMatch && !organisation.inPotentialMatches,
                  'bg-light-yellow': !organisation.correctMatch && organisation.inPotentialMatches
                }"
                class="org-id"
                :value="organisation.expectedId"
                @input="(event) => updateOrganisationId(index, event.target.value)"
              />
            </div>
          </td>
          <td class="pa2">{{ organisation.topMatchName }}</td>
          <td class="pa2">{{ organisation.topMatchId }}</td>
          <td class="pa2 center-align">
            <i v-if="organisation.correctMatch" class="green-text">check_circle</i>
            <i v-if="!organisation.correctMatch" class="red-text">cancel</i>
          </td>
          <td class="pa2 center-align">
            <i v-if="organisation.inPotentialMatches" class="green-text">check_circle</i>
            <i v-if="!organisation.inPotentialMatches" class="red-text">cancel</i>
          </td>
        </tr>
        <tr>
          <td></td>
          <td class="">
            <div class="field small">
              <input class="pa2 bn" v-model="addOrganisationName" placeholder="Organisation name" />
            </div>
          </td>
          <td class="">
            <div class="field small">
              <input class="pa2 bn" v-model="addOrganisationId" placeholder="Expected ID" />
            </div>
          </td>
          <td class="pa2" colspan="4">
            <button @click="addOrganisation"><i>add</i>Add organisation</button>
          </td>
        </tr>
      </tbody>
    </table>
  </main>
</template>

<style scoped></style>
