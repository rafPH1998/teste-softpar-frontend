<template>
  <q-page class="q-pa-md" style="background-color: #e7f1fc; min-height: 100vh;">
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
        <div class="row no-wrap items-center">
          <q-checkbox v-model="task.completed" class="q-mr-sm" color="primary"
            @update:model-value="toggleTaskStatus(task)" />

          <q-input v-model="task.description" dense outlined class="col" readonly
            :style="{ textDecoration: task.completed ? 'line-through' : 'none' }" />
          <q-btn icon="delete" flat round color="red" @click="deleteTask(task.id)" />
        </div>
      </div>

    </div>

    <q-dialog v-model="isModalOpen">
      <q-card>
        <q-card-section>
          <div class="text-h6 text-primary">Adicionar Nova Tarefa</div>
        </q-card-section>
        <q-card-section>
          <q-input v-model="modalData.title" label="Título da Tarefa" outlined dense class="q-my-md" />
          <q-input v-model="modalData.description" label="Descrição" outlined dense type="textarea" />
        </q-card-section>
        <q-card-actions align="right">
          <q-btn flat label="Cancelar" color="secondary" @click="closeModal" />
          <q-btn flat label="Adicionar" :loading="loadingSubmitTask" :disable="loadingSubmitTask" color="primary"
            @click="addTaskFromModal" />
        </q-card-actions>
      </q-card>
    </q-dialog>
  </q-page>
</template>


<script setup lang="ts">
import { ref, computed, onMounted } from 'vue';
import axios from 'axios';


interface Task {
  id: number;
  title: string;
  description: string;
  completed: boolean;
  completedAt?: string | null;
}

const tasks = ref<Task[]>([]);
const isModalOpen = ref<boolean>(false);
const loadingSubmitTask = ref<boolean>(false);
const modalData = ref({ title: '', description: '' });

const apiBaseUrl = 'http://localhost:8085/api/tasks';

onMounted(async () => {
  await fetchTasks()
})

const fetchTasks = async () => {
  try {
    const response = await axios.get(apiBaseUrl);
    tasks.value = response.data.tasks.map((task: Task) => ({
      id: task.id,
      title: task.title,
      description: task.description,
      completed: task.completed,
      completedAt: task.completedAt || null,
    }));

  } catch (error) {
    console.error('Erro ao buscar tarefas:', error);
  }
}

const addTaskFromModal = async () => {
  try {
    loadingSubmitTask.value = true;

    const newTask = {
      title: modalData.value.title,
      description: modalData.value.description,
      completed: false,
    }

    const res = await axios.post(apiBaseUrl, newTask);

    if (res?.status === 201) {
      alert("Tarefa criada com sucesso!")
      tasks.value.push(res.data);
      closeModal();
    }

  } catch (error) {
    console.error('Erro ao adicionar tarefa:', error);
  } finally {
    loadingSubmitTask.value = false;
  }
}


const toggleTaskStatus = async (task: Task) => {
  try {
    await axios.patch(`${apiBaseUrl}/${task.id}/update-status`);
    task.completedAt = task.completed ? new Date().toISOString() : null;
  } catch (error) {
    console.error('Erro ao atualizar status da tarefa:', error);
  }
}

const deleteTask = async (taskId: number) => {
  try {
    await axios.delete(`${apiBaseUrl}/${taskId}`);
    tasks.value = tasks.value.filter(task => task.id !== taskId);
  } catch (error) {
    console.error('Erro ao deletar tarefa:', error);
  }
}

const closeModal = () => {
  modalData.value = { title: '', description: '' };
  isModalOpen.value = false;
}

const openModal = () => {
  isModalOpen.value = true;
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
