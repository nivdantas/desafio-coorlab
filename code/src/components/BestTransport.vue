<template>
  <div class="title">
    <b-navbar toggleable="lg" type="dark" variant="info">
      <b-navbar-brand class="ml-2"> </b-navbar-brand>
    </b-navbar>
    <div class="grid">
      <div class="navfrete">
        <img class="logo" src="../assets/logo.png" alt="" />
        <h1>{{ appName }}</h1>
      </div>

      <form class="formulario" @submit.prevent="filtrarValorFrete">
        <div class="form-group">
          <h2>
            <img class="maplogo" src="../assets/maplogo.svg" alt="" /> Insira o destino
            e o peso
          </h2>
          <label for="city"><b>Cidade de destino:</b></label>
          <!-- Seleção da cidade com o uso de v-model e v-for. -->
          <select id="city" class="form-control" v-model="cidadeSelecionada">
            <option class="first-option" disabled value="">
              Selecione o destino
            </option>
            <option v-for="city in cidades" :key="city">{{ city }}</option>
          </select>
        </div>
        <div class="form-group">
          <!-- Uso de v-model para peso. -->
          <label for="peso"><b>Peso do frete:</b></label>
          <input
            type="number"
            step="any"
            id="peso"
            class="form-control"
            placeholder="Peso da carga em kg"
            v-model="pesoTempoReal"
          />
        </div>
        <button type="submit" class="btn btn-primary" :disabled="estadoApi !== 'pronta'">
          Analisar
        </button>
        <p v-if="estadoApi === 'carregando'" role="status">Carregando destinos…</p>
        <p v-if="mensagem" role="alert">{{ mensagem }}</p>
        <button v-if="estadoApi === 'erro'" type="button" @click="carregarFretes">
          Tentar novamente
        </button>
      </form>
      <div class="resultados" aria-live="polite">
        <div class="semDados" v-if="!freteBarato && estadoApi === 'pronta'">
          <h2>Nenhum dado selecionado.</h2>
        </div>
        <div class="valorFrete" v-if="freteBarato">
          <img src="../assets/freteBarato.svg" alt="" />
          <p><b>Frete com menor valor</b></p>
          <p>Transportadora: {{ freteBarato.name }}</p>
          <p>Tempo de entrega: {{ freteBarato.lead_time }} horas</p>
          <p>Custo total: <b>{{ formatarCustoTotal(freteBarato) }}</b></p>
        </div>
        <div class="valorFrete" v-if="freteRapido">
          <img src="../assets/freteRapido.svg" alt="" />
          <p><b>Frete mais rápido</b></p>
          <p>Transportadora: {{ freteRapido.name }}</p>
          <p>Tempo de entrega: {{ freteRapido.lead_time }} horas</p>
          <p>Custo total: <b>{{ formatarCustoTotal(freteRapido) }}</b></p>
        </div>
        <button v-if="freteBarato" type="button" class="limpar" @click="limparFretes">
          Limpar
        </button>
      </div>
    </div>
  </div>
</template>

<script>
import { BNavbar, BNavbarBrand } from "bootstrap-vue";

const moeda = new Intl.NumberFormat("pt-BR", {
  style: "currency",
  currency: "BRL",
});

function lerNumero(texto, unidade) {
  const valor = texto.replace(unidade, "").trim();
  return valor === "" ? NaN : Number(valor);
}

function tarifaPorPeso(peso) {
  return peso <= 100 ? "cost_transport_light" : "cost_transport_heavy";
}

export default {
  components: {
    BNavbar,
    BNavbarBrand,
  },
  data() {
    return {
      appName: "Melhor Frete",
      pesoTempoReal: null,
      peso: 0,
      estadoApi: "carregando",
      mensagem: "",
      cidadeSelecionada: "",
      cidades: [],
      freteBarato: null,
      freteRapido: null,
      apiData: [],
    };
  },
  created() {
    this.carregarFretes();
  },
  methods: {
    async carregarFretes() {
      this.estadoApi = "carregando";
      this.mensagem = "";

      try {
        const response = await fetch("http://localhost:3000/transport");
        if (!response.ok) throw new Error("Falha na API de fretes");

        const data = await response.json();
        const fretes = data.map((frete) => ({
          ...frete,
          cost_transport_light: lerNumero(frete.cost_transport_light, /^R\$/),
          cost_transport_heavy: lerNumero(frete.cost_transport_heavy, /^R\$/),
          lead_time: lerNumero(frete.lead_time, /h$/),
        }));
        const dadosInvalidos = fretes.some((frete) =>
          typeof frete.name !== "string" || !frete.name.trim() ||
          typeof frete.city !== "string" || !frete.city.trim() ||
          !Number.isFinite(frete.cost_transport_light) || frete.cost_transport_light < 0 ||
          !Number.isFinite(frete.cost_transport_heavy) || frete.cost_transport_heavy < 0 ||
          !Number.isFinite(frete.lead_time) || frete.lead_time < 0
        );
        if (!fretes.length || dadosInvalidos) throw new Error("Dados de frete indisponíveis");

        this.apiData = fretes;
        this.cidades = [...new Set(fretes.map((frete) => frete.city))];
        this.estadoApi = "pronta";
      } catch {
        this.estadoApi = "erro";
        this.mensagem = "Não foi possível carregar os destinos. Verifique se a API está disponível.";
      }
    },
    filtrarValorFrete() {
      if (this.estadoApi !== "pronta") return;

      const peso = Number(this.pesoTempoReal);
      if (!this.cidadeSelecionada || !Number.isFinite(peso) || peso <= 0) {
        this.mensagem = "Selecione uma cidade e insira um peso maior que zero.";
        return;
      }

      const fretes = this.apiData.filter((frete) => frete.city === this.cidadeSelecionada);
      if (!fretes.length) {
        this.mensagem = "Nenhum frete disponível para esta cidade.";
        return;
      }

      const tarifa = tarifaPorPeso(peso);
      fretes.sort((a, b) => a[tarifa] - b[tarifa]);
      this.freteBarato = fretes[0];
      fretes.sort((a, b) => a.lead_time - b.lead_time);
      this.freteRapido = fretes[0];
      this.peso = peso;
      this.mensagem = "";
    },
    formatarCustoTotal(frete) {
      return moeda.format(frete[tarifaPorPeso(this.peso)] * this.peso);
    },
    limparFretes() {
      this.freteRapido = null;
      this.freteBarato = null;
      this.pesoTempoReal = null;
      this.peso = 0;
      this.mensagem = "";
      this.cidadeSelecionada = "";
    },
  },
};
</script>

<style scoped>
/* Estilos aplicados no componente, em ordem de inserção no HTML. */
html {
  background-color: #f2f2f2;
}

h1 {
  font-size: 32px;
}

h2 {
  font-size: 28px;
  margin-bottom: 30px;
  color: #363636;
}

h3 {
  font-size: 24px;
}

label {
  margin-bottom: 10px;
  color: #505059;
}

select {
  background-image: linear-gradient(45deg, transparent 50%, gray 50%),
    linear-gradient(135deg, gray 50%, transparent 50%),
    linear-gradient(to right, #ccc, #ccc);
  background-position: calc(100% - 20px) calc(1em + 2px),
    calc(100% - 15px) calc(1em + 2px), calc(100% - 2.5em) 0.5em;
  background-size: 5px 5px, 5px 5px, 1px 1.5em;
  background-repeat: no-repeat;
}

select:focus {
  background-image: linear-gradient(45deg, green 50%, transparent 50%),
    linear-gradient(135deg, transparent 50%, green 50%),
    linear-gradient(to right, #ccc, #ccc);
  background-position: calc(100% - 15px) 1em, calc(100% - 20px) 1em,
    calc(100% - 2.5em) 0.5em;
  background-size: 5px 5px, 5px 5px, 1px 1.5em;
  background-repeat: no-repeat;
  border-color: green;
  outline: 0;
}

.title .navbar {
  background-color: #596c8a !important;
}

.title .navbar-brand {
  margin-left: 20px;
}
.title {
  height: 100vh;
}
.navbar {
  height: 5vh;
}

.grid {
  height: 80vh;
  margin: 40px auto;
  width: 1500px;
  box-shadow: rgba(100, 100, 111, 0.2) 0px 7px 29px 0px;
  display: grid;
  grid-template-rows: 1fr 14fr;
  grid-template-columns: 1fr 2fr;
  align-items: center;
}

.navfrete {
  display: flex;
  align-items: center;
  grid-column: span 2;
  font-weight: bolder;
  color: white;
  width: 100%;
  height: 6vh;
  background-color: #7195bf;
  border-radius: 10px 10px 0px 0px;
  align-self: start;
}

.navfrete h1 {
  height: 30px;
  color: black;
}

.navfrete h1,
.logo {
  margin-left: 40px;
}
.logo {
  width: 50px;
  height: 50px;
}
.maplogo {
  width: 40px;
  height: 40px;
}
.formulario,
.resultado {
  margin: 40px 20px;
}

.formulario {
  display: flex;
  justify-content: center;
  align-items: center;
  flex-direction: column;
  align-self: flex-end;
  height: 90%;
  width: 500px;
  gap: 40px;
  background: #f2f2f2;
  border-radius: 10px;
}

.formulario input,
.formulario select {
  width: 400px;
}

.formulario button {
  width: 200px;
  background-color: #7195bf;
  color: #363636;
  font-weight: bolder;
  border: none;
}

.resultados {
  display: flex;
  flex-wrap: wrap;
  flex-direction: column;
  gap: 60px;
  justify-self: center;
}

.valorFrete {
  display: grid;
  border-radius: 10px;
  height: 150px;
  width: 800px;
  grid-template-columns: 1fr 4fr;
  grid-template-rows: auto;
  background-color: #5e6c89;
  background-image: linear-gradient(
    to right,
    rgba(0, 0, 0, 0) 20%,
    #e2e2e2 20%
  );
}

.valorFrete img {
  grid-column: 1;
  grid-row: span 4;
  justify-self: center;
  align-self: center;
}

.valorFrete p {
  justify-self: center;
  align-self: center;
  grid-column: 2;
  height: 20px;
}

.valorFrete b {
  font-size: 20px;
}

.first-option {
  color: gray;
}

.limpar {
  width: 200px;
  height: 30px;
  background-color: #626c88;
  color: #363636;
  font-weight: bolder;
  border: none;
  border-radius: 5px;
  align-self: flex-end;
}
</style>
