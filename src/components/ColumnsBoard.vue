<template>
  <el-row class="column-container">
    <el-col
        v-for="(column, columnIndex) in columns"
        :key="column.id"
        :span="8"
        class="column"
    >
      <el-card>
        <h3>{{ column.name }}</h3>
        <el-card
            v-for="(task, taskIndex) in column.tasks"
            :key="task.id"
            class="task"
            :draggable="true"
            @dragstart="onTaskDragStart(columnIndex, taskIndex, $event)"
            @dragover="onDragOver"
            @drop="onDropTask(columnIndex, taskIndex, $event)"
        >
          {{ task.name }}
        </el-card>
      </el-card>
    </el-col>
  </el-row>
</template>

<script>
import { ref, onMounted } from 'vue';
import axios from 'axios';
import { ElRow, ElCol, ElCard } from 'element-plus';

export default {
  components: {
    ElRow,
    ElCol,
    ElCard
  },
  setup() {
    const columns = ref([]);

    const fetchColumns = async () => {
      try {
        const response = await axios.get('http://localhost/api/column');
        columns.value = response.data;
      } catch (error) {
        console.error('Ошибка при загрузке колонок:', error);
      }
    };

    onMounted(fetchColumns);

    let draggedTaskColumnIndex = null;
    let draggedTaskIndex = null;
    let draggedTaskId = null;

    const onTaskDragStart = (columnIndex, taskIndex, event) => {
      draggedTaskColumnIndex = columnIndex;
      draggedTaskIndex = taskIndex;
      draggedTaskId = columns.value[columnIndex].tasks[taskIndex].id;
      event.dataTransfer.effectAllowed = 'move';
    };

    const onDragOver = (event) => {
      event.preventDefault();
    };

    const onDropTask = async (columnIndex, taskIndex, event) => {
      event.preventDefault();
      if (draggedTaskColumnIndex !== null && draggedTaskIndex !== null && draggedTaskId !== null) {
        const sourceColumn = columns.value[draggedTaskColumnIndex];
        const task = sourceColumn.tasks.splice(draggedTaskIndex, 1)[0];
        columns.value[columnIndex].tasks.splice(taskIndex, 0, task);

        await axios.patch(`http://localhost/api/task/${task.id}/move`, {
          target_column_id: columns.value[columnIndex].id,
          target_position: taskIndex + 1
        });

        await axios.patch(`http://localhost/api/column/${sourceColumn.id}/tasks/reorder`, {
          tasks: sourceColumn.tasks.map((task, index) => ({
            id: task.id,
            position: index + 1
          }))
        });

        await axios.patch(`http://localhost/api/column/${columns.value[columnIndex].id}/tasks/reorder`, {
          tasks: columns.value[columnIndex].tasks.map((task, index) => ({
            id: task.id,
            position: index + 1
          }))
        });

        draggedTaskColumnIndex = null;
        draggedTaskIndex = null;
        draggedTaskId = null;
      }
    };

    const moveTask = async (task, direction) => {
      const columnIndex = columns.value.findIndex(col => col.id === task.column_id);
      const taskIndex = columns.value[columnIndex].tasks.findIndex(t => t.id === task.id);
      const newIndex = direction === 'up' ? taskIndex - 1 : taskIndex + 1;

      if (newIndex >= 0 && newIndex < columns.value[columnIndex].tasks.length) {
        const column = columns.value[columnIndex];
        const taskToMove = column.tasks.splice(taskIndex, 1)[0];
        column.tasks.splice(newIndex, 0, taskToMove);

        await axios.patch(`http://localhost/api/task/${task.id}/move`, {
          target_column_id: task.column_id,
          target_position: newIndex + 1
        });

        await axios.patch(`http://localhost/api/column/${column.id}/tasks/reorder`, {
          tasks: column.tasks.map((task, index) => ({
            id: task.id,
            position: index + 1
          }))
        });
      }
    };

    return {
      columns,
      onTaskDragStart,
      onDragOver,
      onDropTask,
      moveTask
    };
  }
};
</script>

<style>
.column-container {
  display: flex;
  flex-wrap: nowrap;
  overflow-x: auto;
}

.column {
  margin: 10px;
}

.task {
  margin: 5px 0;
}

.task-actions {
  margin-top: 10px;
}
</style>
