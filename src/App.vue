<script setup>
import ComponentStatus from "./components/ComponentStatus.vue";
import FormInput from "./components/FormInput.vue";
import { ref } from "vue";
import mqtt from "mqtt";

const showConnection = ref(false);

const values = ref({
  temperature: "0",
  light: "0",
  airQuality: "0",
  proximity: "0",
  humidity: "0",
  fan: "",
});

const enable = ref({
  temperature: false,
  light: false,
  airQuality: false,
  proximity: false,
  fan: false,
});

const connection = ref({
  protocol: "ws",
  host: "broker.emqx.io",
  port: 8083,
  endpoint: "/mqtt",
  clean: true,
  connectTimeout: 30 * 1000,
  reconnectPeriod: 4000,
  clientId: "mqttx_25e34a70",
  username: "",
  password: "",
});

const subscription = ref({
  topic: "F2G7",
  qos: 0,
});

const publish = ref({
  topic: "F2G7",
  qos: 0,
  payload: "",
});

const receiveNews = ref("");

const qosList = [0, 1, 2];

const client = ref({
  connected: false,
});

const subscribeSuccess = ref(false);

const connecting = ref(false);

const retryTimes = ref(0);

const handleOnReConnect = () => {
  retryTimes.value += 1;
  if (retryTimes.value > 5) {
    try {
      client.value.end();
      retryTimes.value = 0;
      connecting.value = false;
    } catch (error) {
      console.log(error);
    }
  }
};

const createConnection = () => {
  connecting.value = true;
  const { protocol, host, port, endpoint, ...options } = connection.value;
  const connectUrl = `${protocol}://${host}:${port}${endpoint}`;
  client.value = mqtt.connect(connectUrl, options);
  if (client.value.on) {
    client.value.on("connect", () => {
      connecting.value = false;
      console.log("Connection succeeded!");
    });
    client.value.on("reconnect", handleOnReConnect);
    client.value.on("error", (error) => {
      console.log("Connection failed", error);
    });
    client.value.on("message", (topic, message) => {
      console.log("Received message", topic, message.toString());
      receiveNews.value = receiveNews.value.concat(message);
      //19,588,71,28,57,0
      if (topic === "F2G7") {
        var data = message.toString().split(",");
        if(data.length !== 6) return;
        values.value.proximity = data[0];
        values.value.light = data[1];
        values.value.airQuality = data[2];
        values.value.temperature = data[3];
        values.value.humidity = data[4];

        if (data[5] === "1") {
          values.value.fan = true;
          enable.value.fan = true;
        } else {
          values.value.fan = false;
          enable.value.fan = false;
        }
      }
    });
  }
};

const doSubscribe = () => {
  const { topic, qos } = subscription.value;
  client.value.subscribe(topic, { qos }, (error, res) => {
    if (error) {
      console.log("Subscribe to topics error", error);
      return;
    }
    subscribeSuccess.value = true;
    console.log("Subscribe to topics res", res);
  });
};

const doUnSubscribe = () => {
  const { topic } = subscription.value;
  client.value.unsubscribe(topic, (error) => {
    if (error) {
      console.log("Unsubscribe error", error);
    }
  });
};

const doPublish = () => {
  enable.value.fan = !enable.value.fan;
  const { topic, qos, payload } = publish.value;
  var pay =  enable.value.fan ? "1" : "0";
  client.value.publish(topic, pay, { qos }, (error) => {
    if (error) {
      console.log("Publish error", error);
    }
    console.log("Publish success");
  });
};

const destroyConnection = () => {
  if (client.value.connected) {
    try {
      client.value.end(false, () => {
        initData();
        console.log("Successfully disconnected!");
      });
    } catch (error) {
      console.log("Disconnect failed", error.toString());
    }
  }
};

const changeFanStatus = (status) => {
  enable.value.fan = status;
};

createConnection();
doSubscribe();
</script>

<template>
  <div class="">
    <div class="flex gap-4 p-8">
      <div>
        <font-awesome-icon
          class="w-10 h-10 text-white"
          :icon="['fab', 'dev']"
        />
      </div>
      <h1 class="text-2xl font-bold text-white text-center">
        Monitoreo de sensores
      </h1>
    </div>
    <div class="flex justify-end max-w-2xl m-auto">
      <button
        class="outline-none cursor-pointer"
        @click="showConnection = !showConnection"
      >
        <font-awesome-icon
          v-if="showConnection"
          class="text-white w-8"
          :icon="['fas', 'angle-down']"
        />
        <font-awesome-icon
          v-else
          class="text-white w-8"
          :icon="['fas', 'angle-up']"
        />
      </button>
    </div>
    <div v-if="showConnection" class="my-4 max-w-4xl m-auto">
      <div class="flex justify-center flex-wrap">
        <FormInput title="Host" v-model="connection.host" />
        <FormInput title="Port" v-model="connection.port" />
        <FormInput title="ClientId" v-model="connection.clientId" />
        <FormInput title="Username" v-model="connection.username" />
        <FormInput title="Password" v-model="connection.password" />
        <FormInput title="Topic" v-model="subscription.topic" />
        <FormInput title="publish" v-model="publish.topic" />
        <FormInput title="Payload" v-model="publish.payload" />
      </div>
      <div class="flex justify-center my-4">
        <button
          class="border border-white rounded-md text-white px-4 py-1"
          @click="createConnection"
        >
          Conectar
        </button>

        <button
          class="border border-white rounded-md text-white px-4 py-1 mx-4"
          @click="destroyConnection"
        >
          Desconectar
        </button>

        <button
          class="border border-white rounded-md text-white px-4 py-1"
          @click="doSubscribe"
        >
          Suscribir
        </button>

        <button
          class="border border-white rounded-md text-white px-4 py-1 ml-4"
          @click="doUnSubscribe"
        >
          Desuscribir
        </button>

        <button
          class="border border-white rounded-md text-white px-4 py-1 ml-4"
          @click="doPublish"
        >
          Publicar
        </button>
      </div>
    </div>

    <div class="p-4 max-w-4xl m-auto">
      <div>
        <div class="grid grid-cols-1 sm:grid-cols-2 gap-8 gap-y-8">
          <ComponentStatus
            title="Temperatura"
            :icon="['fas', 'temperature-quarter']"
            :value="values.temperature"
            :enable="enable.temperature"
          />
          <ComponentStatus
            title="Luz en el ambiente"
            :icon="['fas', 'lightbulb']"
            :value="values.light"
            :enable="enable.light"
          />
          <ComponentStatus
            title="Calidad de aire"
            :icon="['fas', 'wind']"
            :value="values.airQuality"
            :enable="enable.airQuality"
          />
          <ComponentStatus
            title="Proximidad"
            :icon="['fas', 'people-arrows']"
            :value="values.proximity"
            :enable="enable.proximity"
          />
          <ComponentStatus
            title="Humedad"
            :icon="['fas', 'droplet']"
            :value="values.humidity"
            :enable="enable.humidity"
          />
          <div>
            <ComponentStatus
              title="Ventiladores"
              :icon="['fas', 'fan']"
              :value="enable.fan ? 'Encendido' : 'Apagado'"
              :enable="enable.fan"
              size="text-lg"
            />
            <div class="flex justify-center">
              <button
                class="border border-white rounded-md text-white px-4 py-1"
                @click="doPublish"
              >
                <font-awesome-icon
                  class="w-8 h-4"
                  :class="enable.fan ? 'text-green-300' : 'text-red-100'"
                  :icon="['fas', 'power-off']"
                />
              </button>
            </div>
          </div>
        </div>
        <div class="flex justify-center my-8"></div>
      </div>
    </div>
  </div>
</template>

<style scoped></style>
