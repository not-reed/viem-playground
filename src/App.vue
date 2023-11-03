<script setup>
import { createPublicClient, http } from 'viem';
import { computed, ref } from 'vue';
import * as chains from 'viem/chains'
import { Interface } from 'ethers';


function call() {

  const client = createPublicClient({
    chain: chains[network.value],
    transport: http(rpc.value || chains[network.value].rpcUrls.public.http[0])
  })
  return client
}
const rpc = ref('')
const address = ref('')
const blockNumber = ref('')
const abiString = ref('')
const network = ref('mainnet')
const msgSender = ref('')

const abi = computed(() => {
  try {
    return Array.from(new Interface(JSON.parse(abiString.value)).fragments).sort((a, b) => {
      if (a.inputs?.length === b.inputs?.length) return a.name > b.name ? 1 : -1
      if (!a.inputs?.length) return -1;
      if (!b.inputs?.length) return 1
      if (a.name === b.name) return a.inputs.length > b.inputs.length ? 1 : -1
      return a.name > b.name ? 1 : -1
    })
  } catch (e) {
    return []
  }
})

const events = computed(() => abi.value.filter(({ type }) => type === 'event'))
const readFunctions = computed(() => abi.value.filter(({ stateMutability }) => ['view'].includes(stateMutability)))
const writeFunctions = computed(() => abi.value.filter(({ stateMutability }) => ['payable', 'nonpayable'].includes(stateMutability)))


const results = ref({})
const listening = ref({})

async function listenEvent(item) {
  const client = call()

  const contractKey = `${address.value}`
  if (!results.value[contractKey]) {
    results.value[contractKey] = {}
  }

  const itemKey = item.format('sighash')
  if (!results.value[contractKey][itemKey]) {
    results.value[contractKey][itemKey] = []
  }

  const unwatch = client.watchEvent({
    address: address.value,
    event: {
      name: item.name,
      type: item.type,
      anonymous: item.anonymous,
      inputs: item.inputs.map(({ name, type, indexed }) => ({ name, type, indexed })),
    },
    onLogs: logs => {
      logs.forEach(log => {
        console.log(log)
        results.value[contractKey][itemKey].unshift({
          result: JSON.parse(JSON.stringify(log, (key, value) =>
            typeof value === 'bigint'
              ? value.toString()
              : value // return everything else unchanged
          ))
        })
      })
    }
  })

  listening.value[contractKey] = listening.value[contractKey] || {}
  listening.value[contractKey][itemKey] = unwatch
}

async function stopEvent(item) {
  const contractKey = `${address.value}`
  const itemKey = item.format('sighash')


  if (!listening.value[contractKey]?.[itemKey]) {
    return
  }

  listening.value[contractKey][itemKey]()
  delete listening.value[contractKey][itemKey]
}

async function callFunc(e, item) {
  const formData = new FormData(e.target);

  const client = call()

  const opts = {
    abi: [{
      name: item.name,
      stateMutability: item.stateMutability,
      type: item.type,
      inputs: item.inputs.map(({ name, type }) => ({ name, type })),
      outputs: item.outputs.map(({ name, type }) => ({ name, type }))
    }],
    address: address.value,
    functionName: item.name,
  }

  if (item.inputs.length) {
    opts.args = []
    formData.forEach((value, idx) => {
      const input = ['int128', 'uint256'].includes(item.inputs[idx].type) ? BigInt(value) : value
      opts.args.push(input)
    })
  }

  if (msgSender.value) {
    opts.account = msgSender.value
  }

  if (blockNumber.value) {
    opts.blockNumber = Number(blockNumber.value)
  }

  const result = await client.readContract(opts)

  const contractKey = `${address.value}`
  if (!results.value[contractKey]) {
    results.value[contractKey] = {}
  }

  const itemKey = item.format('sighash')
  if (!results.value[contractKey][itemKey]) {
    results.value[contractKey][itemKey] = []
  }
  results.value[contractKey][itemKey].unshift({ inputs: opts.args, result })

  console.log(result)
}

function prettifyAbi() {
  abiString.value = JSON.stringify(JSON.parse(abiString.value), null, 4)
}


</script>

<template>
  <form @submit.prevent="call" class="flex flex-wrap gap-2">
    <div class="w-full">Network Config:</div>

    <select class="px-4 py-2 w-96" v-model="network">
      <optgroup label="Mainnets">
        <option v-for="[symbol, chain] in Object.entries(chains).filter(([, n]) => !n.testnet)" :value="symbol">{{
          chain.name }}
        </option>
      </optgroup>

      <optgroup label="Testnets">
        <option v-for="[symbol, chain] in Object.entries(chains).filter(([, n]) => n.testnet)" :value="symbol">{{
          chain.name }}
        </option>
      </optgroup>
    </select>

    <input :placeholder="chains[network].rpcUrls.public.http[0]" v-model="rpc" class="px-4 py-2" />
    <input placeholder="Block Number" v-model="blockNumber" class="px-4 py-2 w-48" />
    <!-- <label>Use Multicall? <input type="checkbox" v-model="multicall" /></label> -->

    <hr style="width: 100%">
    <div class="w-full pt-4">Contract Config:</div>
    <input placeholder="address" v-model="address" class="w-96 px-4 py-2 my-2" />
    <button type="button" class="bg-zinc-900 py-2" @click="prettifyAbi">Prettify ABI</button>
    <textarea placeholder="abi" v-model=abiString class="w-full h-32 px-4 py-2"></textarea>
    <div v-if="abiString.length && !abi.length" class="text-red-500 w-full">Invalid ABI, Please Check Format. Must be
      parsable JSON</div>


    <button type="submit">Query</button>
  </form>

  <div class="text-left"> Events:
    <div v-for="item in events" class="p-2 flex flex-wrap items-center justify-between bg-zinc-800 my-4 shadow">
      {{ item.format('full') }}
      <button @click="listenEvent(item)" v-if="address && !listening[address]?.[item.format('sighash')]"
        class="ml-4 py-2 w-24 bg-zinc-900">Listen</button>


      <button @click="stopEvent(item)" v-else-if="address && listening[address]?.[item.format('sighash')]"
        class="ml-4 py-2 w-24 bg-zinc-900">Stop</button>


      <div class="bg-zinc-700 px-4 py-2 w-full overflow-auto max-h-96"
        v-if="results[address]?.[item.format('sighash')]?.length">

        <ol v-for="log in results[address][item.format('sighash')]">
          <li class="border border-zinc-900 shadow p-2 my-2">

            <div>

              <div>Tx: {{ log.result.transactionHash }}</div>
              <div>Block: {{ log.result.blockNumber }}</div>

              <ul>
                <li v-for="([key, value]) in Object.entries(log.result.args)">{{ key }}: {{ value }}</li>
              </ul>

            </div>
          </li>

        </ol>

      </div>
    </div>
  </div>

  <div class="text-left"> Read Functions:
    <div v-for="item in readFunctions" class="p-2 flex flex-wrap items-center justify-between bg-zinc-800 my-4 shadow">
      <form @submit.prevent="callFunc($event, item)" class="w-full">
        <div class="flex items-center justify-between w-full">
          {{ item.format('full') }}

          <button v-if="address" type="submit" class="ml-4 py-2 w-24 bg-zinc-900">Call</button>
        </div>
        <div class="flex flex-col w-full" v-if="address">
          <label v-for="(input, inputIdx) in item.inputs" class="flex flex-col">
            <span class="text-sm text-gray-500">
              {{ input.type }} <template v-if="input.name">({{ input.name }})</template>
            </span>
            <input class="p-2" :placeholder="input.type" :name="inputIdx" />
          </label>
        </div>
      </form>

      <div class="bg-zinc-700 px-4 py-2" v-if="results[address]?.[item.format('sighash')]?.length">


        {{ results[address][item.format('sighash')][0].result }}
      </div>
    </div>
  </div>


  <div class="text-left"> Write Functions: <input placeholder="msg.sender" class="px-4 py-2" v-if="address"
      v-model="msgSender" />
    <div v-for="item in writeFunctions" class="p-2 flex flex-wrap items-center justify-between bg-zinc-800 my-4 shadow">
      <form @submit.prevent="callFunc($event, item)" class="w-full">
        <div class="flex items-center justify-between w-full">
          {{ item.format('full') }}

          <button v-if="address" type="submit" class="ml-4 py-2 w-24 bg-zinc-900">Call</button>
        </div>
        <div class="flex flex-col w-full" v-if="address">
          <label v-for="(input, inputIdx) in item.inputs" class="flex flex-col">
            <span class="text-sm text-gray-500">
              {{ input.type }} <template v-if="input.name">({{ input.name }})</template>
            </span>
            <input class="p-2" :placeholder="input.type" :name="inputIdx" />
          </label>
        </div>
      </form>

      <div class="bg-zinc-700 px-4 py-2" v-if="results[address]?.[item.format('sighash')]?.length">{{
        results[address][item.format('sighash')][0].result }}</div>
    </div>
  </div>
</template>

