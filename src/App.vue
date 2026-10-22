<script setup lang="ts">
import { ref } from 'vue'
import axios from 'axios'
import {
  Card,
  InputNumber,
  Button,
  FileUpload,
  ProgressSpinner,
  MeterGroup,
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

  await axios
    .post(
      'https://api.team-backend.hack.week-infra.ru/api/v1/predict/by-id',
      JSON.stringify({ index: String(objId.value) }),
      {
        headers: { 'Content-Type': 'application/json' },
      },
    )
    .then((response) => {
      const _data: ByIdResponse = response.data
      predictions.value = [_data]
      showError.value = false
      errorMessage.value = ''
    })
    .catch((error) => {
      if (error.response && error.response.status === 404) {
        errorMessage.value = 'ID объекта не найден'
      } else {
        errorMessage.value = 'Произошла ошибка при расчете'
      }
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

        const _data: CsvResponse = response.data
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
        } else {
          errorMessage.value = 'Произошла ошибка при расчете'
        }
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
                    <MeterGroup
                      :value="[
                        {
                          label: '',
                          value: convertValue(prediction.probability),
                          color: 'var(--p-primary-color)',
                        },
                      ]"
                    ></MeterGroup>
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
  box-sizing: border-box;
  min-height: 100vh;
  padding: clamp(1rem, 3vw, 2rem);
  display: flex;
  justify-content: center;
}

.content {
  width: min(100%, 960px);
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 2rem 1rem;
  align-content: start;
}

.content > * {
  min-width: 0;
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
  grid-column: 1 / -1;
}

.card-content {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.control-content {
  height: 100%;
  flex-direction: row;
  justify-content: space-between;
  align-items: center;
}

.control-fileupload {
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
  grid-template-columns: minmax(7rem, 1fr) minmax(0, 4fr);
  gap: 1rem 0.5rem;
}

.result-grid > * {
  min-width: 0;
  overflow-wrap: anywhere;
}

.result-grid-title {
  font-weight: 500;
  line-height: 20px;
}

@media (max-width: 640px) {
  .container {
    padding: 0.75rem;
  }

  .content {
    grid-template-columns: minmax(0, 1fr);
    gap: 1rem;
  }

  .title {
    font-size: 1.25rem;
    line-height: 1.3;
  }

  .subtitle {
    line-height: 1.4;
  }

  .control-content {
    flex-direction: column;
    align-items: stretch;
  }

  .control-content > *,
  .control-content .p-button,
  .control-fileupload .p-fileupload,
  .control-fileupload .p-button {
    width: 100%;
  }

  .control-fileupload {
    align-items: stretch;
  }

  .result-grid {
    grid-template-columns: minmax(5.5rem, 1fr) minmax(0, 2fr);
    font-size: 0.875rem;
  }
}
</style>
