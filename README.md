<h1 align="center">Desafio Coorlab</h1>
Desafio proposto pela Coorlab na criação de um mecanismo para determinar o frete mais econômico e o mais rápido de uma API com os dados necessários.
Selecione a cidade de destino e informe o peso da carga em kg. O simulador compara a opção mais econômica e a mais rápida. A tarifa por kg muda para cargas acima de 100 kg. Em caso de empate no tempo de entrega, é exibida a opção com menor custo.

# ✔️ Tecnologias utilizadas
- `Vue.JS`
- `JavaScript`
- `HTML e CSS`

# 🛠️ Abrir e rodar o projeto
Na raiz do repositório, instale as dependências e inicie a API local:

```sh
cd code
npm ci
npm run api_serve
```

Em outro terminal, execute `npm run serve` dentro de `code/` e abra a URL exibida pelo comando. O frontend consulta `http://localhost:3000/transport`, servido a partir de `code/data.json`.

Na pasta `code/`, execute `npm run lint -- --no-fix` para verificar o código e `npm run build` para gerar os arquivos em `code/dist/`.

![Desafio Coorlab GIF](https://i.imgur.com/Wfb60Mc.gif)
