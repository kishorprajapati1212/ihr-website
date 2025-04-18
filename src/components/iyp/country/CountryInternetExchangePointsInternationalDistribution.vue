<script setup>
import { useRoute, useRouter } from 'vue-router'
import { ref, inject, watch, onMounted } from 'vue'
import IypGenericTable from '@/components/tables/IypGenericTable.vue'
import IypGenericBarChart from '@/components/charts/IypGenericBarChart.vue'
import IypGenericHeatmapChart from '@/components/charts/IypGenericHeatmapChart.vue'
import { QSlider, QBadge } from 'quasar'

const iyp_api = inject('iyp_api')

const props = defineProps(['countryCode', 'pageTitle'])

const route = useRoute()
const router = useRouter()

const ixps = ref({
  data: [],
  group: {},
  show: false,
  loading: true,
  query: `MATCH (member:AS)-[mem:MEMBER_OF]-(ix:IXP)-[:COUNTRY]-(ix_country:Country {country_code:$country_code})
    MATCH (member:AS)-[:COUNTRY {reference_org:'NRO'}]-(as_country:Country)
    WHERE  as_country.country_code <> $country_code AND (member)-[:ORIGINATE]-(:Prefix)
    OPTIONAL MATCH (member)-[:CATEGORIZED {reference_name:'bgptools.as_names'}]-(tag:Tag)
    OPTIONAL MATCH (ix:IXP)-[:MANAGED_BY {reference_org:'PeeringDB'}]-(org:Organization)
    RETURN  member.asn AS asn, coalesce(tag.label, 'Other') AS label, ix.name AS ix_name, ix_country.country_code AS ix_country, as_country.country_code AS as_country, mem.reference_org AS mem_reference_org, org.name AS org_name`,
  columns: [
  {
      name: 'ASN',
      label: 'ASN',
      align: 'left',
      field: (row) => row.asn,
      format: (val) => `AS${val}`,
      sortable: true,
    },
    {
      name: 'Organization',
      label: 'Organization',
      align: 'left',
      field: (row) => row.org_name,
      format: (val) => `${val}`,
      sortable: true,
    },
    {
      name: 'IXP Name',
      label: 'IXP Name',
      align: 'left',
      field: (row) => row.ix_name,
      format: (val) => `${val}`,
      sortable: true,
    },
    {
      name: 'ASN Country',
      label: 'ASN Country',
      align: 'left',
      field: (row) => row.as_country,
      format: (val) => `${val}`,
      sortable: true,
    },
    {
      name: 'IXP Country',
      label: 'IXP Country',
      align: 'left',
      field: (row) => row.ix_country,
      format: (val) => `${val}`,
      sortable: true,
    },
    {
      name: 'IXP Label',
      label: 'IXP Label',
      align: 'left',
      field: (row) => row.label,
      format: (val) => `${val}`,
      sortable: true,
    },
    {
      name: 'Reference Organization',
      label: 'Reference Organization',
      align: 'left',
      field: (row) => row.mem_reference_org,
      format: (val) => `${val}`,
      sortable: true,
    }
  ],
  pagination: {
    sortBy: 'Number of members', //string column name
    descending: true //boolean
  }
})
const uniqueASperIXP = ref(6)
const uniqueASperIXPMax = ref(0)
const uniqueASperIXPMin = ref(0)

const load = () => {
  ixps.value.loading = true
  // Run the cypher query
  let query_params = { country_code: props.countryCode }
  iyp_api.run([{ statement: ixps.value.query, parameters: query_params }]).then((results) => {
    ixps.value.data = results[0]
    ixps.value.group = results[0].reduce((acc, current) => {
      if (current.ix_name) {
        const key = `${current.ix_name.toLowerCase()} - ${current.ix_country}`
        if (!acc[key]) {
          acc[key] = new Set()
        }
        if (current.asn) {
          acc[key].add(current.asn)
        }
      }
      return acc
    }, {})
    const ixpsSize = Object.values(ixps.value.group).map(obj => obj.size).sort((a, b) => b - a)
    uniqueASperIXPMax.value = ixpsSize[0]
    const uniqueIxpSize = new Set(ixpsSize)
    if (uniqueIxpSize.size > 5) {
      uniqueASperIXP.value = Array.from(uniqueIxpSize)[4]
    } else {
      uniqueASperIXP.value = Array.from(uniqueIxpSize)[uniqueIxpSize.size - 1]
    }
    ixps.value.loading = false
  })
}

const barPlotDataFormat = (data) => {
  const filteredIXPs = new Set()
  Object.keys(ixps.value.group).forEach(ixp => {
    if (ixps.value.group[ixp].size >= uniqueASperIXP.value) {
      filteredIXPs.add(ixp.split(' - ')[0])
    }
  })

  const countryToASN = {}
  data.forEach(obj => {
    if (filteredIXPs.has(obj.ix_name.toLowerCase())) {
      if (!countryToASN[obj.as_country]) {
        countryToASN[obj.as_country] = new Set()
      }
      countryToASN[obj.as_country].add(obj.asn)
    }
  })

  const asnCountry = []
  Object.keys(countryToASN).forEach(cc => {
    countryToASN[cc].forEach(_ => {
      asnCountry.push({
        as_country: cc
      })
    })
  })

  return asnCountry
}

const heatmapPlotDataFormat= (data) => {
  const ixpDistribution = {}
  Object.keys(data).forEach(ixp => {
    if (data[ixp].size >= uniqueASperIXP.value) {
      ixpDistribution[ixp] = data[ixp]
    }
  })
  return [ixpDistribution]
}

watch(
  () => props.countryCode,
  () => {
    load()
  }
)

onMounted(() => {
  load()
})
</script>

<template>
  <IypGenericTable
    :data="ixps.data"
    :columns="ixps.columns"
    :loading-status="ixps.loading"
    :cypher-query="ixps.query.replace(/\$(.*?)}/, `'${countryCode}'}`)"
    :pagination="ixps.pagination"
    :slot-length="1"
  >
    <div class="col-6">
      <div>
        <h3>Filtering</h3>
        <QBadge>Number of unique ASes per IXP: {{ uniqueASperIXP }}</QBadge>
        <QSlider v-model="uniqueASperIXP" :min="uniqueASperIXPMin" :max="uniqueASperIXPMax" />
      </div>
      <IypGenericBarChart
        v-if="ixps.data.length > 0"
        :chart-data="barPlotDataFormat(ixps.data)"
        :chart-layout="{ title: 'Top countries where ASNs peer (nb. unique ASNs)', yaxis: { title: { text: 'Number of peers' } } }"
        :config="{ key: 'as_country' }"
        :group-top-n-and-except-as-others="5"
      />
      <IypGenericHeatmapChart
        v-if="ixps.data.length > 0"
        :chart-data="heatmapPlotDataFormat(ixps.group)"
        :chart-layout="{ title: 'IXPs distribution', xaxis: { automargin: true }, yaxis: { automargin: true } }"
        :config="{ }"
      />
    </div>
  </IypGenericTable>
</template>
