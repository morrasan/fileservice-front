<script setup>
  import { ref } from 'vue'
  import axios from 'axios'

  const fileUploadFormForm = ref(null);
  const isFormValid = ref(false)

  const userName = ref('')
  const userEmail = ref('')
  const userFiles = ref([])

  const isStoreError = ref(false)

  const rules = {
    userName: [
      v => !!v || 'Imię jest wymagane',
      v => (v && v.length > 2) || 'Podaj więcej 2 znaków',
    ],
    userEmail: [
      v => !!v || 'E-mail jest wymagane',
      v => /^[a-zA-Z0-9.!#$%&’*+\/=?^_`\{\|\}~\-]+@[a-zA-Z0-9\-]+(?:\.[a-zA-Z0-9\-]+)+$/.test(v) || 'Adres e-mail ma nieprawidłowy format',
    ],
    userFiles: [
      v => !!v.length || 'Przynajmniej jeden plik wymagany',
      v => fileSizesRule(v),
      v => fileImageResolutionsRule(v),
    ]
  }

  /**
   * Single file size validation: no more than 5 mb
   * @param file
   * @returns {boolean}
   */
  const validateFileSize = (file) => {
    const fileSizeMB = file.size / (1024 * 1024)
    return fileSizeMB <= 5
  }

  /**
   * Single file resolution validation: not less than 500x500
   * @param file
   * @returns {Promise<unknown>}
   */
  const validateImageResolution = (file) => {
    return new Promise((resolve, reject) => {
      const image = new Image()
      image.src = URL.createObjectURL(file)

      image.onload = () => {
        const imageWidth = image.width
        const imageHeight = image.height
        resolve(imageWidth >= 500 && imageHeight >= 500)
      }

      image.onerror = () => {
        reject('Incorrect file format');
      }
    })
  }

  /**
   * Multiple file sizes validation
   * @param files
   * @returns {boolean|string}
   */
  const fileSizesRule = (files) => {
    const wrongSizeFiles = []
    files.forEach((file) => {
      if (!validateFileSize(file)) {
        wrongSizeFiles.push(file.name)
      }
    })

    if (wrongSizeFiles.length) {
      return `Maksymalny rozmiar plików 5MB nie spełnia się dla ${wrongSizeFiles.join(', ')}.`
    }
    return true
  }

  /**
   * Multiple file resolutions validation
   * @param files
   * @returns {Promise<boolean|string>}
   */
  const fileImageResolutionsRule = async (files) => {
    const wrongResolutionFiles = []
    for (let i = 0; i < files.length; i++) {
      const file = files[i]
      try {
        const resolutionValid = await validateImageResolution(file)
        if (!resolutionValid) {
          wrongResolutionFiles.push(file.name)
        }
      } catch (error) {
        wrongResolutionFiles.push(file.name)
      }
    }

    if (wrongResolutionFiles.length) {
      return `Minimalna rozdzielczość plików 500x500 nie spełnia się dla ${wrongResolutionFiles.join(', ')}.`
    }
    return true
  }

  const storeData = () => {

    isStoreError.value = false

    // run validation
    fileUploadFormForm.value.validate()

    if (isFormValid.value) {

      // Prepare post data
      const formData = new FormData()
      formData.append('name', userName.value)
      formData.append('email', userEmail.value)

      const files = userFiles.value;

      for (let i = 0; i < files.length; i++) {
        formData.append('files[]', files[i]);
      }

      // get API url from .env
      const url = import.meta.env.VITE_API_URL

      axios.post(url + '/api/files', formData, {
        headers: {
          'Content-Type': 'multipart/form-data',
          'Accept': 'application/json'
        }

      }).then(() => {

        // set default values
        userName.value = ''
        userEmail.value = ''
        userFiles.value = []

        fileUploadFormForm.value.resetValidation()

        // fetch new data from back to list in FileList component
        document.dispatchEvent(new CustomEvent('fetchData'))

      }).catch(error => {

        isStoreError.value = true
        console.error('error', error);
      })
    }
  }
</script>

<template>
  <h3 class="text-center mb-3">Załaduj swoje pliki</h3>
  <v-alert
      v-if="isStoreError"
      color="error"
      class="ma-3"
      closable
  >
    Coś poszło nie tak, spróbuj póżniej
  </v-alert>
  <v-form
    ref="fileUploadFormForm"
    v-model="isFormValid"
  >
    <v-text-field
      ref="nameInput"
      v-model.trim="userName"
      :rules="rules.userName"
      variant="outlined"
      rounded
      label="Name"
      class="mb-2"
    ></v-text-field>
    <v-text-field
      ref="emailInput"
      v-model.trim="userEmail"
      :rules="rules.userEmail"
      variant="outlined"
      rounded
      label="E-mail"
      class="mb-2"
    ></v-text-field>
    <v-file-input
      ref="fileInput"
      v-model="userFiles"
      :rules="rules.userFiles"
      accept="image/jpeg, image/png, image/webp, image/tiff, image/bmp"
      label="Pliki"
      variant="outlined"
      rounded
      chips
      counter
      show-size
      multiple
      class="mb-2"
    ></v-file-input>
    <v-alert
      title="Ładujemy tylko takie pliki:"
      type="info"
      variant="tonal"
      class="mb-3"
    >
      <template #text>
        <ul>
          <li>Dozwolone typy plików: JPG, PNG, WebP, TIFF, BMP</li>
          <li>Maksymalny rozmiar pliku: 5 MB</li>
          <li>Minimalna rozdzielczość pliku: 500 x 500 px</li>
        </ul>
      </template>
    </v-alert>
  </v-form>
  <v-btn
    block
    rounded="xl"
    color="green"
    size="x-large"
    @click="storeData"
  >
    Ładuj
  </v-btn>

</template>

<style scoped>
.v-alert {
  border-radius: 24px;
}
</style>
