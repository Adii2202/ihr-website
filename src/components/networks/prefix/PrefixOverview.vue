<script setup>
import { QChip, QSpinner, QMarkupTable } from 'quasar'
import { RouterLink } from 'vue-router'
import Tr from '@/i18n/translation'
import { ref, inject, watch, onMounted } from 'vue'
import '@/styles/chart.sass'

const iyp_api = inject('iyp_api')

const REFERENCES = {
  'bgp.he.net': 'https://bgp.he.net/net',
  'bgp.tools': 'https://bgp.tools/prefix',
  'stat.ripe.net': 'https://stat.ripe.net/app/launchpad',
}

const props = defineProps({
  host: {
    type: String,
    required: true,
  },
  prefixLength: {
    type: Number,
    required: true,
  },
  external: {
    type: Boolean,
    required: false,
    default: false,
  },
  getPrefix: {
    type: String,
    required: true
  }
})

const loading = ref(2)
const references = ref(REFERENCES)
const queries = ref([
  {
    data: [],
    query: `MATCH (p:Prefix {prefix: $prefix})
      OPTIONAL MATCH (p)<-[o:ORIGINATE]-(a:AS)
      OPTIONAL MATCH (a)-[:NAME {reference_org:'PeeringDB'}]->(pdbn:Name)
      OPTIONAL MATCH (a)-[:NAME {reference_org:'BGP.Tools'}]->(btn:Name)
      OPTIONAL MATCH (a)-[:NAME {reference_org:'RIPE NCC'}]->(ripen:Name)
      OPTIONAL MATCH(p)-[deleg:COUNTRY {reference_name: 'nro.delegated_stats'}]->(c:Country)
      OPTIONAL MATCH (p)-[:CATEGORIZED]->(t:Tag)
      RETURN p.prefix AS prefix, head(collect(DISTINCT(o.descr))) AS descr, collect(DISTINCT([toString(a.asn), COALESCE(pdbn.name, btn.name, ripen.name)])) AS asn, c.name AS country, collect(DISTINCT(t.label)) AS tags, deleg.registry AS rir, c.country_code AS cc`
  },
  {
    data: [],
    query: `MATCH (p:Prefix {prefix: $prefix})<-[:PART_OF]-(:IP)<-[:RESOLVES_TO]-(h:HostName)
      OPTIONAL MATCH (h)-[:PART_OF]-(:DomainName)-[ra:RANK]->(:Ranking {name: 'Tranco top 1M'})
      RETURN  DISTINCT h.name as hostname, ra.rank AS rank ORDER BY rank LIMIT 5`
  }
])

const fetchData = async () => {
  let params = { prefix: props.getPrefix }
  let results = await iyp_api.run(queries.value.map(obj => ({statement: obj.query, parameters: params})))

  queries.value[0].data = results[0]
  loading.value -= 1
  queries.value[1].data = results[1]
  loading.value -= 1
}

const handleReference = (key) => {
  let externalLink = ''
  if (key === 'bgp.he.net') {
    externalLink = `${references.value[key]}/${props.host}/${props.prefixLength}`
  } else if (key === 'bgp.tools') {
    externalLink = `${references.value[key]}/${props.host}/${props.prefixLength}`
  } else if (key === 'stat.ripe.net') {
    externalLink = `${references.value[key]}/${props.host}/${props.prefixLength}`
  } else {
    console.log('none')
    return
  }
  return externalLink
}

watch(() => props.host, () => {
  loading.value = 3
  queries.value.forEach( query => {
    query.data = []
  })
  fetchData()
})

watch(() => props.prefixLength, () => {
  loading.value = 3
  queries.value.forEach( query => {
    query.data = []
  })
  fetchData()
})

onMounted(() => {
  fetchData()
})
</script>

<template>
  <div>
    <QMarkupTable separator="horizontal">
      <div v-if="loading > 0" class="IHR_loading-spinner">
        <QSpinner color="secondary" size="15em"/>
      </div>
      <thead>
        <tr>
          <th class="text-left">Summary</th>
          <th class="text-left">Originated By</th>
          <th class="text-left">Popular Hostnames</th>
          <th class="text-left">External Links</th>
        </tr>
      </thead>
      <tbody>
        <tr>
          <td class="text-left">
            <div v-if="queries[0].data.length > 0">
              <div v-if="queries[0].data[0].country">
                Registered in
                <RouterLink :to="Tr.i18nRoute({ name: 'countries', params: {cc:queries[0].data[0].cc } })">{{ queries[0].data[0].country }}</RouterLink>
                ({{ queries[0].data[0].rir.toUpperCase() }})
              </div>
            </div>
          </td>
          <td>
            <div v-if="queries[0].data.length > 0">
              <div v-if="queries[0].data[0].asn[0][0]">
                <div v-for="item in queries[0].data[0].asn" :key='item[0]'>
                  <RouterLink :to="Tr.i18nRoute({ name:'networks', params:{ id:`AS${item[0]}` } })">
                    AS{{ item[0] }} {{ item[1] }}
                  </RouterLink>
                </div>
              </div>
              <div v-else>
                <p>Prefix Not Announced on BGP</p>
              </div>
            </div>
          </td>
          <td class="text-left">
            <div v-if="queries[1].data.length > 0">
              <div v-for="item in queries[1].data" :key="item.hostname">
                <RouterLink :to="Tr.i18nRoute({ name: 'hostnames', params: {hostName:item.hostname}})">
                    {{ item.hostname }}
                  </RouterLink>
              </div>
            </div>
          </td>
          <td class="text-left">
            <div v-if="queries[0].data.length > 0">
              <div v-for="(value, key) in references" :key="key">
                <a :href="handleReference(key)" target="_blank" rel="noreferrer">
                  {{ key }}
                </a>
              </div>
            </div>
          </td>
        </tr>
      </tbody>
    </QMarkupTable>
    <br />
    <QMarkupTable>
      <thead>
        <tr>
          <th class="text-left" :colspan="5">Tags</th>
        </tr>
      </thead>
      <tbody>
        <tr>
          <td :colspan="5">
            <div  v-if="queries[0].data.length > 0" class="row">
              <RouterLink v-for="tag in queries[0].data[0].tags" :key="tag" :to="Tr.i18nRoute({ name: 'tags', params: {tag: tag}, hash: '#Prefixes'})">
                <QChip dense size="md" color="info" text-color="white">{{ tag }}</QChip>
              </RouterLink>
            </div>
          </td>
        </tr>
      </tbody>
    </QMarkupTable>
  </div>
</template>

<style lang="stylus">
p {
  font-size: 1rem;
  margin-bottom: 0;
}
</style>