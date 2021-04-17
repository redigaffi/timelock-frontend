<template>
  <span v-if="this.connectedAddress != false"><strong>Connected Address <br />{{this.connectedAddress}}</strong></span>
  <h1>Timelock MastherChef Caller &#128512;</h1>
  <hr />
  
  <button @click="exportState">Export state</button>
  <input type="text" placeholder="Previous state..." v-model="importStateData" /> 
  <button @click="importState">Import state</button>

  <h2>Timelock configuration</h2>
  <div class="container">
    <label>Timelock contract </label>
    <input type="text" v-model="timelockAddress"/>
    
    <label>MasterChefV2 contract </label>
    <input type="text" v-model="masterChefAddress"/>

    <label>Gas Limit </label>
    <input type="text" v-model="gasLimit"/>

    <label>Value  (as string, eg: 0.5) </label>
    <input type="text" v-model="value"/>

    <label>ETA (timestamp - BSC new block minted every 3s approx) </label>
    <input type="text" v-model="timestamp" readonly/>
    <button @click="updateEta(6)">now + 6 hours</button>
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
        <label>{{ param.type }} | {{ param.name }}</label>
        <input type="" v-model="param.val" />
      </div>
      <button v-on:click="queueTransaction">Queue Transaction</button>
      <button v-on:click="executeTransaction">Execute Transaction</button>
    </div>
  </div>
</template>

<script>
const { ethers } = require("ethers");
const Timelock = require("./abi/Timelock.json");
const MasterChefV2 = require("./abi/MasterChefV2.json");
const provider = new ethers.providers.JsonRpcProvider(window.ethereum);

export default {
  name: "App",
  components: {},
  beforeMount: async function() {
    await window.ethereum.enable()
  
    this.provider = new ethers.providers.Web3Provider(window.ethereum);
    this.signer = this.provider.getSigner();
    this.connectedAddress = await this.signer.getAddress();
    
    const timeLock = new ethers.Contract(
        this.timelockAddress,
        Timelock.abi,
        provider
    );
    this.timeLockWithSigner = timeLock.connect(this.signer);

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
      timeLockWithSigner: false,
      connectedAddress: false,
      provider: false,
      signer: false,
      gasLimit: 6721975,
      value: 0,
      functionParams: {
        add: [],
        updateEmissionRate: []
      },
      functionDescription: {
        add: "Add a new staking/farming pool to masterchef",
        updateEmissionRate: "Tokens minted per block"
      },
      functionToCall: "",
      timelockAddress: "0xE3c56475C0B5be384E660A13f6168A6d6eb79c5C",
      masterChefAddress: "0x83A1B359C7559eC0225e7b6436cA124EB5F85b5A",
      timestamp: 0,
      importStateData: "",
    };
  },
  methods: {
    exportState: function() {
      console.log(JSON.stringify({
        gasLimit: this.gasLimit,
        value: this.value,
        functionParams: this.functionParams,
        functionToCall: this.functionToCall,
        timelockAddress: this.timelockAddress,
        masterChefAddress: this.masterChefAddress,
        timestamp: this.timestamp
      }));
    },

    importState: function() {
      const state = JSON.parse(this.importStateData);
      this.gasLimit = state.gasLimit;
      this.value = state.value;
      this.functionParams = state.functionParams;
      this.functionToCall = state.functionToCall;
      this.timelockAddress = state.timelockAddress;
      this.masterChefAddress = state.masterChefAddress;
      this.timestamp = state.timestamp;
    },

    updateEta: function(time) {
      this.timestamp = Date.now() + (time*60);
    },

    joinMethodParamSignature: function(name, params) {
      let signature = name + "(";

      for (let param in params) {
        let p = params[param];
        signature += p.type + " " + p.name + ", ";
      }
      signature += ");"
      return signature;
    },

    joinFunctionParametersAndGetEncodedBytes: function(functionToCall) {
      let iface = new ethers.utils.Interface(MasterChefV2.abi);
      let params = [];

      for (let key in this.functionParams[this.functionToCall]) {
        const p = this.functionParams[this.functionToCall][key];
        
        if (p.type === "bool") {
          p.val = (p.val === "true");
        }

        params.push(p.val);
      }

      return iface.encodeFunctionData(functionToCall, params);
    },
    
    queueTransaction: async function() {
      const overrides = {
        // To convert Ether to Wei:
        //value: ethers.utils.parseEther("0.5"), // ether in this case MUST be a string
        gasLimit: 6721975,
      };

      const bytes = this.joinFunctionParametersAndGetEncodedBytes(this.functionToCall);

      let tx = await this.timeLockWithSigner.queueTransaction(
        this.masterChefAddress,
        0,
        '',
        bytes,
        this.timestamp,
        overrides
      );

      console.log(tx);
    },

    executeTransaction: async function() {
      const overrides = {
        // To convert Ether to Wei:
        value: ethers.utils.parseEther("0.5"), // ether in this case MUST be a string
        gasLimit: 6721975,
      };

      const bytes = this.joinFunctionParametersAndGetEncodedBytes(this.functionToCall);

      let tx = await this.timeLockWithSigner.executeTransaction(
        this.masterChefAddress,
        0,
        '',
        bytes,
        this.timestamp,
        overrides
      );

      console.log(tx);
    },
  }
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
