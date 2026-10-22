<script setup lang="ts">
import { ref } from 'vue'
import axios from 'axios'
import {
  Card,
  InputNumber,
  Button,
  FileUpload,
  ProgressBar,
  ProgressSpinner,
  type FileUploadSelectEvent,
} from 'openvue'

interface ByIdResponse {
  index: string
  probability: number
}

interface CsvResponse {
  predictions: ByIdResponse[]
}

const objId = ref<number | undefined>()
const loading = ref<boolean>(false)
const showError = ref<boolean>(false)
const showResult = ref<boolean>(false)
const errorMessage = ref<string>('')

const predictions = ref<ByIdResponse[]>([])

const calcById = async () => {
  if (!objId.value) {
    return
  }
  showResult.value = true
  loading.value = true

  predictions.value = [{ index: String(objId.value), probability: 0.265987646 }]
  loading.value = false

  await axios
    .post(
      'https://api.team-backend.hack.week-infra.ru/api/v1/predict/by-id',
      JSON.stringify({ index: String(objId.value) }),
      {
        headers: { 'Content-Type': 'application/json' },
      },
    )
    .then((response) => {
      const _data: ByIdResponse = JSON.parse(response.data)
      predictions.value = [_data]
      showError.value = false
      errorMessage.value = ''
    })
    .catch((error) => {
      if (error.response && error.response.status === 404) {
        errorMessage.value = 'ID объекта не найден'
      }
      errorMessage.value = 'Произошла ошибка при расчете'
      showError.value = true
    })
    .finally(() => (loading.value = false))
}

const loadCsv = async (event: FileUploadSelectEvent) => {
  if (event.files) {
    showResult.value = true
    loading.value = true

    const file = event.files[0]
    const formData = new FormData()
    formData.append('file', file)

    await axios
      .post('https://api.team-backend.hack.week-infra.ru/api/v1/predict/csv', formData)
      .then((response) => {
        showError.value = false
        errorMessage.value = ''

        const _data: CsvResponse = JSON.parse(response.data)
        if (_data.predictions.length === 0) {
          errorMessage.value = 'Произошла ошибка при расчете'
          showError.value = true
          predictions.value = []
        } else {
          predictions.value = _data.predictions
        }
      })
      .catch((error) => {
        if (error.response && error.response.status === 413) {
          errorMessage.value = 'Файл слишком большой'
        }
        errorMessage.value = 'Произошла ошибка при расчете'
        showError.value = true
      })
      .finally(() => (loading.value = false))
  } else {
    console.log('event.files is empty')
  }
}

const convertValue = (value: number): number => {
  return Math.round(value * 100)
}
</script>

<template>
  <div class="container">
    <div class="content">
      <div class="full-width">
        <div class="title">Оценка вероятности банкротства физлица</div>
        <div class="subtitle">
          Для рассчета укажите ID объекта из датасета или загрузите csv файл
        </div>
      </div>

      <Card>
        <template #subtitle>ID объекта</template>
        <template #content>
          <div class="card-content control-content">
            <InputNumber v-model="objId" allow-empty :use-grouping="false" fluid></InputNumber>
            <div>
              <Button label="Рассчитать" :disabled="loading || !objId" @click="calcById"></Button>
            </div>
          </div>
        </template>
      </Card>

      <Card>
        <template #subtitle>Загрузить CSV</template>
        <template #content>
          <div class="control-fileupload">
            <FileUpload
              :disabled="loading"
              mode="basic"
              name="file[]"
              accept=".csv"
              custom-upload
              invalid-file-size-message="	{0}: Размер файла должен быть меньше {1}"
              choose-label="Загрузить"
              choose-icon="oi oi-upload"
              :choose-button-props="{ outlined: true }"
              @select="loadCsv"
            ></FileUpload>
          </div>
        </template>
      </Card>

      <Card v-if="showResult" class="full-width">
        <template #title>Вероятность банкротства</template>
        <template #content>
          <div class="result-content">
            <template v-if="loading">
              <ProgressSpinner
                style="width: 100px; height: 100px"
                stroke-width="4"
              ></ProgressSpinner>
              <div>Данные обрабатываются</div>
            </template>
            <template v-else-if="showError">
              <i class="oi oi-times-circle error-icon"></i>
              <div>{{ errorMessage }}</div>
            </template>
            <template v-else>
              <div class="result-grid">
                <div class="result-grid-title">ID объекта</div>
                <div class="result-grid-title">Оценка риска моделью</div>

                <template v-for="prediction in predictions" :key="prediction.index">
                  <div>{{ prediction.index }}</div>
                  <div>
                    <ProgressBar
                      :value="convertValue(prediction.probability)"
                      v-tooltip="{
                        value: `${convertValue(prediction.probability)}%`,
                        disabled: convertValue(prediction.probability) > 2,
                      }"
                    ></ProgressBar>
                  </div>
                </template>
              </div>
            </template>
          </div>
        </template>
      </Card>
    </div>
  </div>
</template>

<style>
.container {
  padding: 1rem;
  display: flex;
  justify-content: center;
}

.content {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 2rem 1rem;
}

.title {
  font-size: 1.5rem;
  font-weight: 500;
  line-height: 24pt;
  letter-spacing: 0.15px;
}

.subtitle {
  font-size: 0.875rem;
  font-weight: 400;
  line-height: 20pt;
}

.full-width {
  grid-column: 1 / 3;
}

.card-content {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.control-content {
  height: 100%;
  min-width: 380px;
  flex-direction: row;
  justify-content: space-between;
  align-items: center;
}

.control-fileupload {
  min-width: 380px;
  display: flex;
  flex-direction: row;
  justify-content: space-between;
  align-items: end;
  gap: 2rem;
}

.result-error {
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  gap: 1rem;
}

.result-content {
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  gap: 1rem;
}

.result-content-column {
  display: flex;
  justify-content: center;
  align-items: center;
  flex-direction: column;
}

.error-icon {
  font-size: 3rem;
  color: #dc2626;
}

.result-grid {
  padding-top: 1rem;
  width: 100%;
  display: grid;
  grid-template-columns: 1fr 4fr;
  gap: 1rem 0.5rem;
}

.result-grid-title {
  font-weight: 500;
  line-height: 20px;
}
</style>
