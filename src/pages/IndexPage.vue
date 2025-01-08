<template>
  <q-page class="q-pa-md" style="background-color: #e7f1fc; min-height: 100vh;">
    <div class="q-mb-md bg-white q-pa-md">
      <!-- Botão de Filtro (visível apenas no mobile) -->
      <q-btn class="q-mb-md" label="Filtrar" color="primary" icon="filter_list" @click="dialogFilter = true"
        v-show="isMobile" />

      <!-- Filtros para desktop -->
      <div class="q-row q-gutter-md" style="display: flex; justify-content: space-between;" v-show="!isMobile">
        <div class="q-col" style="flex: 1;">
          <q-select v-model="filters.status" :options="['Concluídas', 'Pendentes']" label="Status"
            @input="applyFilters" />
        </div>

        <div class="q-col" style="flex: 1;">
          <q-input v-model="filters.dateStart" label="Data Inicial" mask="date" type="date" @input="applyFilters" />
        </div>

        <div class="q-col" style="flex: 1;">
          <q-input v-model="filters.dateEnd" label="Data Final" mask="date" type="date" @input="applyFilters" />
        </div>

        <div class="q-col" style="flex: 1;">
          <q-select v-model="filters.orderBy" :options="['Título', 'Data']" label="Ordenar por" @input="applyFilters" />
        </div>

        <div class="q-col-auto">
          <q-btn icon="delete" label="Limpar Filtros" color="red" flat @click="clearFilters" />
        </div>
      </div>

      <!-- Filtros (Mobile) -->
      <q-dialog v-model="dialogFilter">
        <q-card>
          <q-card-section>
            <div class="text-h6">Filtros</div>
          </q-card-section>

          <q-card-section>
            <q-select v-model="filters.status" :options="['Concluídas', 'Pendentes']" label="Status"
              @input="applyFilters" />
            <q-input v-model="filters.dateStart" label="Data Inicial" mask="date" type="date" @input="applyFilters"
              class="q-mt-md" />
            <q-input v-model="filters.dateEnd" label="Data Final" mask="date" type="date" @input="applyFilters"
              class="q-mt-md" />
            <q-select v-model="filters.orderBy" :options="['Título', 'Data']" label="Ordenar por" @input="applyFilters"
              class="q-mt-md" />
          </q-card-section>

          <q-card-actions align="right">
            <q-btn flat label="Fechar" color="primary" @click="dialogFilter = false" />
            <q-btn flat label="Limpar Filtros" color="red" @click="clearFilters" />
          </q-card-actions>
        </q-card>
      </q-dialog>
    </div>

    <div class="row justify-center q-mb-md">
      <q-card class="q-pa-md text-center col-3 q-mx-sm">
        <q-icon name="check_circle" size="2rem" color="green" />
        <div class="text-h6">Concluídas</div>
        <div class="text-subtitle1">{{ completedTasks }}</div>
      </q-card>
      <q-card class="q-pa-md text-center col-3 q-mx-sm">
        <q-icon name="warning" size="2rem" color="orange" />
        <div class="text-h6">Não Concluídas</div>
        <div class="text-subtitle1">{{ incompleteTasks }}</div>
      </q-card>
      <q-card class="q-pa-md text-center col-3 q-mx-sm">
        <q-icon name="format_list_bulleted" size="2rem" color="blue" />
        <div class="text-h6">Total</div>
        <div class="text-subtitle1">{{ tasks.length }}</div>
      </q-card>
    </div>

    <div class="todo-container q-pa-lg bg-white shadow-2 rounded text-center" style="max-width: 600px; margin: auto;">
      <h3 class="text-primary">TODOLIST</h3>

      <q-linear-progress :value="progress" color="green" size="10px" rounded class="q-my-md">
        {{ Math.round(progress * 100) }}%
      </q-linear-progress>

      <q-btn icon="add" color="primary" label="Adicionar Tarefa" @click="openModal" class="q-my-md" />

      <div v-for="task in tasks" :key="task.id" class="q-my-md task-item" style="transition: transform 0.3s;">
        <h5 class="text-left text-primary">{{ task.title }}</h5>
        <p v-if="task.completed" class="text-left text-grey q-my-sm">
          Completado em: {{ task.completed_at }}
        </p>
        <div class="row no-wrap items-center">
          <q-checkbox v-model="task.completed" class="q-mr-sm" color="primary"
            @update:model-value="toggleTaskStatus(task)" />

          <q-input v-model="task.description" dense outlined class="col" readonly
            :style="{ textDecoration: task.completed ? 'line-through' : 'none' }" />

          <q-btn icon="edit" :disable="task.completed" flat round color="primary" @click="editTask(task)" />
          <q-btn icon="delete" :disable="task.completed" flat round color="red" @click="confirmDeleteTask(task.id)" />

          <!-- modal para deletar uma tarefa-->
          <q-dialog v-model="dialogVisible">
            <q-card>
              <q-card-section>
                <div class="text-h6">Tem certeza que deseja excluir esta tarefa?</div>
              </q-card-section>
              <q-card-actions>
                <q-btn flat label="Cancelar" color="primary" @click="dialogVisible = false" />
                <q-btn flat label="Sim" color="negative" @click="deleteTaskConfirmed" />
              </q-card-actions>
            </q-card>
          </q-dialog>
        </div>
      </div>

    </div>

    <q-dialog v-model="isModalOpen">
      <q-card>
        <q-card-section>
          <div class="text-h6 text-primary">{{ messageModal }}</div>
        </q-card-section>
        <q-card-section>
          <q-input v-model="modalData.title" label="Título da Tarefa" outlined dense class="q-my-md" />
          <q-input v-model="modalData.description" label="Descrição" outlined dense type="textarea" />
        </q-card-section>
        <q-card-actions align="right">
          <q-btn flat label="Cancelar" color="secondary" @click="closeModal" />
          <q-btn flat label="Salvar" :loading="loadingSubmitTask" :disable="loadingSubmitTask" color="primary"
            @click="addOrEditTask" />
        </q-card-actions>
      </q-card>
    </q-dialog>
  </q-page>
</template>

<script setup lang="ts">
import { ref, computed, onMounted, watch } from 'vue';
import axios from 'axios';
import { useQuasar } from "quasar";

interface Task {
  id: number;
  title: string;
  description: string;
  completed: boolean;
  completed_at?: string | null;
}

const $q = useQuasar();
const isMobile = computed(() => $q.screen.lt.sm);

const tasks = ref<Task[]>([]);

const isModalOpen = ref<boolean>(false);
const isEditModel = ref<boolean>(false);
const modalData = ref<{ id?: number; title: string; description: string }>({ title: '', description: '' });

const loadingSubmitTask = ref<boolean>(false);
const dialogVisible = ref<boolean>(false);
const dialogFilter = ref<boolean>(false);

const taskToDelete = ref<number | null>(null);
const messageModal = ref<string | null>(null);

const filters = ref<{
  status: 'Concluídas' | 'Pendentes' | '';
  orderBy: 'Título' | 'Data' | '';
  dateStart: string;
  dateEnd: string;
}>({
  status: '',
  orderBy: '',
  dateStart: '',
  dateEnd: '',
});

const apiBaseUrl = 'http://localhost:8085/api/tasks';

onMounted(async () => {
  await fetchTasks()
})

const statusMapping: Record<'Concluídas' | 'Pendentes', string> = {
  'Concluídas': 'completed',
  'Pendentes': 'pending',
}

const orderByMapping: Record<'Título' | 'Data', string> = {
  'Título': 'title',
  'Data': 'created_at',
}

const fetchTasks = async () => {
  const mappedStatus = statusMapping[filters.value.status as keyof typeof statusMapping] || '';
  const orderMapping = orderByMapping[filters.value.orderBy as keyof typeof orderByMapping] || '';

  try {
    const response = await axios.get(apiBaseUrl, {
      params: {
        status: mappedStatus,
        order_by: orderMapping,
        date_start: filters.value.dateStart,
        date_end: filters.value.dateEnd,
      }
    });
    tasks.value = response.data.tasks.map((task: Task) => ({
      id: task.id,
      title: task.title,
      description: task.description,
      completed: task.completed,
      completed_at: task.completed_at || null,
    }));
  } catch (error) {
    console.error('Erro ao buscar tarefas:', error);
  }
}

watch(filters, () => {
  fetchTasks();
}, { deep: true });

const applyFilters = async () => {
  await fetchTasks();
}

const clearFilters = async () => {
  filters.value = {
    status: '',
    orderBy: '',
    dateStart: '',
    dateEnd: '',
  };
  await fetchTasks();
}

const addOrEditTask = async () => {

  try {
    loadingSubmitTask.value = true;

    const taskData = {
      title: modalData.value.title,
      description: modalData.value.description,
      completed: false
    };

    if (isEditModel.value && modalData.value.id) {

      const res = await axios.put(`${apiBaseUrl}/${modalData.value.id}`, taskData);
      if (res?.status === 200) {
        const taskIndex = tasks.value.findIndex(task => task.id === modalData.value.id);
        if (taskIndex !== -1) {
          tasks.value[taskIndex] = res.data;
        }
        alert("Tarefa editada com sucesso!");
        closeModal();
      }
    } else {
      const res = await axios.post(apiBaseUrl, taskData);
      if (res?.status === 201) {
        alert("Tarefa criada com sucesso!");
        tasks.value.push(res.data);
        closeModal();
      }
    }
  } catch (error) {
    console.error('Erro ao adicionar ou editar tarefa:', error);
  } finally {
    loadingSubmitTask.value = false;
  }
};

const editTask = (task: Task) => {
  messageModal.value = "Editar tarefa"
  isEditModel.value = true;
  modalData.value = { ...task };
  isModalOpen.value = true;
}

const toggleTaskStatus = async (task: Task) => {
  try {
    await axios.patch(`${apiBaseUrl}/${task.id}/update-status`);
    task.completed_at = task.completed ? new Date().toISOString() : null;
  } catch (error) {
    console.error('Erro ao atualizar status da tarefa:', error);
  }
}

const confirmDeleteTask = (taskId: number) => {
  taskToDelete.value = taskId;
  dialogVisible.value = true;
}

const deleteTaskConfirmed = async () => {
  try {
    if (taskToDelete.value !== null) {
      await axios.delete(`${apiBaseUrl}/${taskToDelete.value}`);
      tasks.value = tasks.value.filter(task => task.id !== taskToDelete.value);
      dialogVisible.value = false;
    }
  } catch (error) {
    console.error('Erro ao deletar tarefa:', error);
  }
}

const closeModal = () => {
  modalData.value = { title: '', description: '' };
  isModalOpen.value = false;
  isEditModel.value = false;
  messageModal.value = ""
}

const openModal = () => {
  isModalOpen.value = true;
  messageModal.value = "Adicionar nova tarefa"
}

const progress = computed(() => {
  const totalTasks = tasks.value.length;
  const completedTasks = tasks.value.filter(task => task && task.completed).length;
  return totalTasks > 0 ? completedTasks / totalTasks : 0;
})

const completedTasks = computed(() => tasks.value.filter(task => task.completed).length);
const incompleteTasks = computed(() => tasks.value.filter(task => !task.completed).length);
</script>



<style scoped>
.todo-container {
  background-color: white;
  border-radius: 8px;
  padding: 16px;
}

.task-list-container:hover {
  transform: translateY(-10px);
  box-shadow: 0 4px 10px rgba(0, 0, 0, 0.2);
}

.task-item:hover {
  transform: scale(1.03);
  background-color: #f4f9ff;
  border-radius: 8px;
}
</style>