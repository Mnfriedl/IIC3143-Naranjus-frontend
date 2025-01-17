<template>
  <div class="container">
    <div class="row">
      <div class="col-md-12">
        <h2>Vista en vivo de la camara</h2>
        Camara actual: <code v-if="device">{{ device.label }}</code>
        <div class="frame">
          <vue-web-cam
            ref="camera"
            width="80%"
            height="80%"
            :device-id="deviceId"
            @error="onError"
            @cameras="onCameras"
            @camera-change="onCameraChange"
          />
        </div>
        <div class="row">
          <div class="col-md-12">
            <select v-model="camera">
              <option>-- Select Device --</option>
              <option
                v-for="device in devices"
                :key="device.deviceId"
                :value="device.deviceId">
                {{ device.label }}
              </option>
            </select>
          </div>
        </div>
        <div class="row">
          <div class="flex">
            <img :src="imgSource" width="60px" height="180px" class="image">
              <div class="flex-column">
                <h3 class="left-margin">{{imgText[0]}}</h3>
                <h3 class="left-margin">{{imgText[1]}}</h3>
              </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { WebCam } from "vue-web-cam";
export default {
    name: "App",
    components: {
        "vue-web-cam": WebCam
    },
    data() {
        return {
            img: null,
            camera: null,
            deviceId: null,
            devices: [],
            current_plate: "",
            access: 'Esperando',
        };
    },
    mounted: function () {
      window.setInterval(() => {
        this.onCapture()
      }, 3000)
    },
    computed: {
        device: function() {
          return this.devices.find(n => n.deviceId === this.deviceId);
        },
        imgSource() { 
          var images = require.context('../assets/', false, /\.jpeg$/)
          if (this.access == 'Permitido') {
            return images('./semaforo_' + "verde" + ".jpeg")  
          } else if (this.access == "Esperando") {
            return images('./semaforo_' + "amarillo" + ".jpeg")  
          } else {
            return images('./semaforo_' + "rojo" + ".jpeg")
          }
        },
        imgText() {
          if (this.access == 'Permitido') {
            return ["Accesso Permitido", `La patente es ${this.current_plate}`]
          } else if (this.access == "Negado") {
            return ["Accesso Denegado", `La patente es ${this.current_plate}`] 
          } else if (this.access == "Ya realizado") {
            return ["Accesso Denegado, esta visita ya ingresó", `La patente es ${this.current_plate}`] 
          } else {
            return ["Esperando que llegue un vehículo", ""]
          }
        }
    },
    watch: {
        camera: function(id) {
            this.deviceId = id;
        },
        devices: function() {
            // Once we have a list select the first one
            const [first, ...tail] = this.devices;
            if (first) {
                this.camera = first.deviceId;
                this.deviceId = first.deviceId;
            }
        }
    },
    methods: {
        onCapture() {
            this.img = this.$refs.camera.capture().split(',')[1];
            this.plateRecognition(this.img)
        },
        onError(error) {
            console.log("Ocurrio el siguiente error intentando obtener la imagen: ", error);
        },
        onCameras(cameras) {
            this.devices = cameras;
        },
        onCameraChange(deviceId) {
            this.deviceId = deviceId;
            this.camera = deviceId;
        },
        async plateRecognition(img) {
          const data = { "image": img }
          try {
            const res = await fetch(`${process.env.VUE_APP_BACKEND}/plates/check/`, {
              method: 'post',
              cache: 'no-cache',
              mode: 'cors',
              headers: {
                'Content-Type': 'application/json',
                'Origin': 'http://localhost:8080',
                'Authorization': "Bearer " + localStorage.access,
              },
              body: JSON.stringify(data)
            });
            const body = await res.json();
            this.current_plate = body.recognized_text;
            if (body.msg == "Visit was done"){
              this.access = "Ya realizado"
            } else if (body.msg == "Accepted") {
              this.access = "Permitido"
            } else if (body.msg ==  "Plate doesn't have visit registration") {
              this.access = "Negado"
            } else {
              this.access = "Esperando"
            }
          } catch (e) {
            this.access = "Esperando"
            this.current_plate = ""
          }
        }
    }
};
</script>

<style>
.flex {
  display: flex;
  flex-direction: row;
  align-items: center;
}
.frame {
  width: '80%';
  height: '80%';
  border: 3px solid #ccc;
}
.left-margin {
  margin-left: 40px;
  }
.flex-column {
  display: flex;
  direction: column;
}
.image {
  margin-left: 280px;
  margin-top: 40px;
}

</style>