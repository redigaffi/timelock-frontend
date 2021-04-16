<template>
  <h1>Timelock MastherChef Caller &#128512;</h1>

  <h2>Timelock configuration</h2>
  <div class="container">
    <lable>Timelock contract: </lable>
    <input type="text" v-model="timelockAddress"/>
    
    <lable>MasterChefV2 contract: </lable>
    <input type="text" v-model="masterChefAddress"/>

    <lable>Gas Limit: </lable>
    <input type="text" v-model="gasLimit"/>

    <lable>Value  (as string, eg: 0.5): </lable>
    <input type="text" v-model="value"/>

    <lable>ETA (timestamp - BSC new block minted every 3s approx): </lable>
    <input type="text" v-model="timestamp"/>
  </div>

  <h2>Masterchef function</h2>

  <br />

  <div class="container">
    <select name="function" v-model="functionToCall">
      <option
        v-for="(params, name) in this.functionParams"
        :key="name"
        :value="name"
      >
        {{ this.joinMethodParamSignature(name, params) }}
      </option>
    </select>
    <div v-if="functionToCall">
      <p v-if="functionDescription[functionToCall]">
        <strong>Description:</strong> {{ functionDescription[functionToCall] }}
      </p>
      <br />

      <h2>
        {{ functionToCall.charAt(0).toUpperCase() + functionToCall.slice(1) }}
        function parameters
      </h2>
      <div v-for="param in functionParams[functionToCall]" :key="param.name">
        <lable>{{ param.type }} | {{ param.name }}</lable>
        <input type="" v-model="param.val" />
      </div>
      <button v-on:click="sendBlockchain">Send Transaction</button>
    </div>
  </div>
</template>

<script>
const { ethers } = require("ethers");
const Timelock = require("./abi/Timelock.json");
const MasterChefV2 = require("./abi/MasterChefV2.json");
const provider = new ethers.providers.JsonRpcProvider("http://localhost:7545");

export default {
  name: "App",
  components: {},
  beforeMount: function() {
    for (let obj in MasterChefV2.abi) {
      const entry = MasterChefV2.abi[obj];
      if (
        entry.type !== "function" ||
        !Object.keys(this.functionParams).includes(entry.name)
      ) {
        continue;
      }

      this.functionParams[entry.name] = entry.inputs;
    }
  },
  data: () => {
    return {
      gasLimit: 6721975,
      value: 0,
      functionParams: {
        add: [],
        updateEmissionRate: []
      },
      functionDescription: {
        add: "Add a new staking/farming pool to masterchef",
      },
      functionToCall: "",
      timelockAddress: "0x33bDf05A756c413595da9f4F5Be49953BCE641c1",
      masterChefAddress: "0x9Df2D690953B373F027549D6B245551381e9dbF0",
      timestamp: 1,
    };
  },
  methods: {
    joinMethodParamSignature: function(name, params) {
      let signature = name + "(";

      for (let param in params) {
        let p = params[param];
        signature += p.type + " " + p.name + ", ";
      }

      return signature;
    },

    sendBlockchain: async function() {
      const signer0 = provider.getSigner(0);

      const overrides = {
        // To convert Ether to Wei:
        value: ethers.utils.parseEther("0.5"), // ether in this case MUST be a string
        gasLimit: 6721975,
      };

      const timeLock = new ethers.Contract(
        this.timelockAddress,
        Timelock.abi,
        provider
      );
      const timeLockWithSigner = timeLock.connect(signer0);

      let iface = new ethers.utils.Interface(MasterChefV2.abi);
      let params = [];

      //@todo: update
      for (let key in this.functionParams[this.functionToCall]) {
        const p = this.functionParams[this.functionToCall][key];
        params.push(p.val);
      }

      let bytes = iface.encodeFunctionData("add", params);
      let tx = await timeLockWithSigner.executeTransaction(
        this.masterChefAddress,
        0,
        '',
        bytes,
        this.timestamp,
        overrides
      );
      console.log(tx);
    },
  },
};
</script>

<style>
.container {
  width: 450px;
  margin: auto;
}

.container input, select{
  width: 100%;
  margin-bottom: 15px;
}
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
</style>
