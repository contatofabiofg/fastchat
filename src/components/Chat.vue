<script setup lang="ts">
import { ref, type Ref, onBeforeUnmount, watch, onMounted } from 'vue'
import { io } from 'socket.io-client'

interface Mensagem {
  tipo: string
  nome: string
  nomeantigo: string
  texto: string
  hash: string
  dataEHora: string
}

const mensagens: Ref<Mensagem[]> = ref([])
const mensagem = ref('')
const nome = ref('')
const nomeantigo = ref('')
const usuario = ref('')
const retorno: Ref<Mensagem> = ref({} as Mensagem)
const socket = io(import.meta.env.VITE_BASE_URL)
const hash = new Date().getTime().toString()
const telaDeMensagens = ref(document.querySelector('#mensagens'))
const conectado = ref(false)
const objDesconectado = ref({
  tipo: 'conversa',
  hash: '0',
  nome: 'Fast Chat',
  nomeantigo: '',
  texto:
    'Você está desconectado do servidor; conecte-se para interagir com pessoas reais. Obrigado!',
  dataEHora: new Date().toString(),
})

setInterval(() => {
  if (socket.connected) {
    conectado.value = true
  } else {
    conectado.value = false
  }
}, 1000)

watch(retorno, (obj: Mensagem): void => {
  mensagens.value.push(obj)
  setTimeout(() => {
    telaDeMensagens.value!.scrollTop = telaDeMensagens.value!.scrollHeight
  }, 50)
})

async function enviarMensagem() {
  let obj = {
    tipo: 'conversa',
    hash: hash,
    nome: usuario.value,
    nomeantigo: '',
    texto: mensagem.value,
    dataEHora: new Date().toString(),
  }
  if (conectado.value) {
    socket.emit('mensagem', obj)
    socket.on('retorno', async (msg) => {
      retorno.value = await msg
    })
  } else {
    retorno.value = obj
    setTimeout(() => {
      retorno.value = objDesconectado.value
    }, 600)
  }

  mensagem.value = ''
}

function entrarNaConversa() {
  if (usuario.value == '') {
    usuario.value = nome.value
    nome.value = ''
    let obj: Mensagem = {
      tipo: 'join',
      nome: usuario.value,
      hash: hash,
      nomeantigo: '',
      texto: '',
      dataEHora: new Date().toString(),
    }
    if (conectado.value) {
      socket.emit('mensagem', obj)
      socket.on('retorno', async (msg) => {
        retorno.value = await msg
      })
    } else {
      retorno.value = obj
    }
  } else {
    nomeantigo.value = usuario.value
    usuario.value = nome.value
    nome.value = ''
    let obj: Mensagem = {
      tipo: 'change',
      nome: usuario.value,
      hash: hash,
      nomeantigo: nomeantigo.value,
      texto: '',
      dataEHora: new Date().toString(),
    }
    if (conectado.value) {
      socket.emit('mensagem', obj)
      socket.on('retorno', async (msg) => {
        retorno.value = await msg
      })
    } else {
      retorno.value = obj
    }
  }
}

function disconnect() {
  if (socket) {
    socket.disconnect()
  }
}

onMounted(() => {
  telaDeMensagens.value = document.querySelector('#mensagens')
})

onBeforeUnmount(() => {
  disconnect()
})
</script>

<template>
  <div class="w-screen h-screen flex justify-center items-center">
    <div
      class="w-[90vw] lg:w-[50vw] h-[90vh] drop-shadow-md border border-slate-200 bg-white rounded-md p-3 outline-none"
    >
      <div class="w-full flex justify-between items-center my-2">
        <h1 class="font-bold">fast chat</h1>
        <div class="flex gap-2">
          <div
            v-if="conectado"
            class="w-fit self-end px-2 border border-slate-300 rounded-full bg-slate-100 text-[10px]"
          >
            Servidor conectado
            <div
              class="inline-block ml-2 bg-green-500 w-2 h-2 rounded-full"
            ></div>
          </div>
          <div
            v-else
            class="w-fit self-end px-2 border border-slate-300 rounded-full bg-slate-100 text-[10px]"
          >
            Servidor offline
            <div
              class="inline-block ml-2 bg-red-500 w-2 h-2 rounded-full"
            ></div>
          </div>
          <div
            v-if="usuario != ''"
            class="w-fit self-end px-2 border border-slate-300 rounded-full bg-slate-100 text-[10px]"
          >
            Logado como: <span>{{ usuario }}</span>
            <div
              class="inline-block ml-2 bg-green-500 w-2 h-2 rounded-full"
            ></div>
          </div>
          <div
            v-else
            class="w-fit self-end px-2 border border-slate-300 rounded-full bg-slate-100 text-[10px]"
          >
            Offline
            <div
              class="inline-block ml-2 bg-red-500 w-2 h-2 rounded-full"
            ></div>
          </div>
        </div>
      </div>
      <input
        type="text"
        v-model="nome"
        class="w-full border border-slate-300 my-2 mb-4 p-2 rounded-md outline-none"
        :placeholder="
          usuario == ''
            ? 'Digite seu nome para entrar na sala'
            : 'Altere o seu nome'
        "
        @keypress.enter="entrarNaConversa"
      />

      <div
        id="mensagens"
        class="w-full h-[70%] border border-slate-300 rounded-md overflow-auto p-2 bg-[url('/src/assets/background.jpg')]"
      >
        <div
          v-for="(mensagem, index) in mensagens"
          :key="index"
          class="flex flex-col"
        >
          <div
            v-if="mensagem.tipo == 'join'"
            class="rounded-lg drop-shadow w-fit bg-gradient-to-t from-green-200 to-green-100 text-[12px] px-2 py-1 my-3 mx-auto"
          >
            <span class="italic text-gray-400"
              >{{ mensagem.nome }} entrou na conversa</span
            >
          </div>
          <div
            v-else-if="mensagem.tipo == 'change'"
            class="rounded-lg drop-shadow w-fit bg-gradient-to-t from-green-200 to-green-100 text-[12px] px-2 py-1 my-3 mx-auto"
          >
            <span class="italic text-gray-400"
              >{{ mensagem.nomeantigo }} mudou seu nome para
              {{ mensagem.nome }}</span
            >
          </div>
          <div
            v-else
            class="rounded-lg drop-shadow w-fit p-2 pb-4 my-3 relative text-[14px] max-w-[80%]"
            :class="
              mensagem.hash == hash
                ? 'bg-gradient-to-t from-blue-200 to-blue-100 self-end'
                : 'bg-gradient-to-t from-gray-200 to-gray-100'
            "
          >
            <span class="font-bold">{{ mensagem.nome }}</span
            >: {{ mensagem.texto }}
            <span class="absolute bottom-1 right-1 text-[10px] text-gray-500">{{
              new Date(mensagem.dataEHora).getHours() +
              ':' +
              new Date(mensagem.dataEHora).getMinutes()
            }}</span>
          </div>
        </div>
      </div>
      <div
        id="inputArea"
        class="flex relative justify-center items-center mt-5"
      >
        <input
          type="text"
          class="w-full border border-slate-300 rounded-md p-2 pl-4 outline-none mr-2"
          v-model="mensagem"
          :disabled="usuario == ''"
          @keypress.enter="enviarMensagem"
        />
        <button
          class="absolute right-0 opacity-75 hover:opacity-100 w-10 h-10 rounded-full"
          @click="usuario != '' ? enviarMensagem() : undefined"
        >
          <img src="../assets/send-message.png" alt="" class="w-6" />
        </button>
      </div>
    </div>
  </div>
</template>

<style scoped></style>
