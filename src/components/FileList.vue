<script setup>
import { ref, onBeforeMount, onBeforeUnmount } from 'vue'
import axios from "axios";

const cursor = ref(0)

const items = ref([])

const isDeleteError = ref(false)

// set listener for fetch new uploaded items
onBeforeMount(() => {
  document.addEventListener('fetchData', fetchDataByCursor)
})

onBeforeUnmount(() => {
  document.removeEventListener('fetchData', fetchDataByCursor)
})

// fetch data from back
const fetchDataByCursor = ({ done }) => {

  const url = import.meta.env.VITE_API_URL

  axios.get(url + '/api/files', {
    params: {
      cursor: cursor.value
    },
    headers: {
      'Accept': 'application/json'
    }
  }).then(result => {

    if (result.data.files instanceof Array) {

      const filesData = result.data.files.map(item => {
        return {
          id: item.id,
          filename: item.filename,
          filesize: (item.filesize / 1024).toFixed(2),
          extension: item.extension,
          resolution: item.resolution,
          ownerName: item.ownerName,
          thumbnailUrl: url + '/' + item.thumbnailUrl,
          downloadUrl: url + '/api/files/' + item.id
        }
      })

      if (result.data.nextCursor) {

        // add received files to the list
        items.value.push(...filesData)

        // set cursor on the last item
        cursor.value = result.data.nextCursor

        done('ok')

      } else {

        done('empty')
      }
    } else {

      done('empty')
    }

  }).catch(error => {

    console.error('ERROR:', error)
    done('error')
  })
}

// delete item on back and remove from list
const deleteItem = (fileId) => {

  isDeleteError.value = false

  const url = import.meta.env.VITE_API_URL

  axios.delete(url + `/api/files/${fileId}`, {
    headers: {
      'Accept': 'application/json'
    }

  }).then(() => {

    // remove item from the list
    const indexToRemove = items.value.findIndex(item => item.id === fileId);

    if (indexToRemove !== -1) {

      items.value.splice(indexToRemove, 1);
    }

  }).catch(error => {

    console.error('Delete error', error)

    isDeleteError.value = true
  })

}
</script>

<template>
  <h3 class="text-center mb-3">Lista załadowanych plików</h3>
  <v-alert
    v-if="isDeleteError"
    color="error"
    class="ma-3"
    closable
  >
    Coś poszło nie tak, spróbuj póżniej
  </v-alert>
  <v-infinite-scroll :height="400" :items="items" :onLoad="fetchDataByCursor">
    <template v-for="(item, index) in items" :key="item">
      <div :class="['pa-6', index % 2 === 0 ? 'bg-grey-lighten-2' : '']">
        <div class="d-flex flex-row">
          <div>
            <v-img
              width="120"
              cover
              :src="item.thumbnailUrl"
            ></v-img>
          </div>
          <div class="ml-7">
            <ul>
              <li class="long-word">Nazwa pliku: {{ item.filename }}</li>
              <li>Właściciel: {{ item.ownerName }}</li>
              <li>Rozszerzenie: {{ item.extension }}</li>
              <li>Rozmiar pliku, Kb: {{ item.filesize }}</li>
              <li>Rozdielczość: {{ item.resolution }}</li>
            </ul>
          </div>
        </div>
        <div class="d-flex flex-row align-center justify-space-between mt-3">
          <a :href="item.downloadUrl">
            <v-btn color="blue">Pobierz plik</v-btn>
          </a>
          <v-btn
            color="red"
            @click="deleteItem(item.id)"
          >
            Usuń plik
          </v-btn>
        </div>
      </div>
    </template>
  </v-infinite-scroll>
</template>

<style scoped>
.long-word {
  word-break: break-all;
}
.v-alert {
  border-radius: 24px;
}
</style>
