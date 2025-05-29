# to_do_list_project
# to_do_list_project

## 项目架构整体规划
1. **后端（Django）**
    - 用 Django 构建 RESTful API，为前端提供数据服务。
    - ORM 连接 MySQL 数据库，存储任务数据。
2. **前端（Vue.js）**
    - 使用 Vue.js 构建动态界面，展示任务列表，添加、修改、删除任务。
    - 前端通过 Axios 与后端 API 通信，实现功能交互。
3. **数据库（MySQL）**
    - 表结构设计：`Task` 表存储任务数据（任务 ID、标题、状态、创建时间）。
4. **部署**
    - **后端**：使用 Docker 容器化部署 Django 后端服务，配合 Gunicorn 运行。
    - **前端**：打包 Vue.js 项目并通过 Docker 容器化部署，使用 Nginx 作为静态文件服务器。
    - **数据库**：将 MySQL 部署到 Docker 容器中。

## 项目目录结构
```
to_do_list_project/
├── backend/                # 后端 Django 服务
│   ├── manage.py           # Django 管理文件
│   ├── backend/            # Django 项目目录
│   ├── tasks/              # 应用：任务管理
│   │   ├── models.py       # 任务模型定义
│   │   ├── serializers.py  # 序列化器定义
│   │   ├── views.py        # 视图函数定义
│   │   └── urls.py         # 路由定义
│   └── requirements.txt    # 后端依赖文件
├── frontend/               # 前端 Vue.js 项目
│   ├── public/
│   ├── src/
│   │   ├── components/     # 组件目录
│   │   │   └── TaskManager.vue # 任务管理组件
│   │   ├── router/         # 路由配置目录
│   │   │   └── index.js    # 路由配置文件
│   │   ├── plugins/        # 插件目录
│   │   │   └── axios.js    # Axios 配置文件
│   │   └── main.js         # 项目入口文件
│   ├── package.json        # 前端依赖文件
│   └── package-lock.json
├── Dockerfile.backend      # 后端 Dockerfile
├── Dockerfile.frontend     # 前端 Dockerfile
├── docker-compose.yml      # Docker Compose 配置文件
├── README.md               # 项目说明文件
└── ...
```

## 总体步骤概览
1. [准备开发环境](#step1)
    - 安装必要工具（Linux、VSCode、Python、Node.js、MySQL 等）。
2. [后端 Django 开发](#step2)
    - 搭建 Django 后端。
    - 连接 MySQL 数据库，创建 API。
3. [前端 Vue.js 开发](#step3)
    - 创建动态任务界面。
    - 通过 Axios 与后端交互。
4. [部署项目](#step4)
    - 使用 Docker 和 Docker Compose 容器化部署前后端和数据库。

<a id="step1"></a>
## 第一部分：准备开发环境
### 1. 安装必要工具
#### 1.1 更新系统并安装基本依赖
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install build-essential curl git -y
```
#### 1.2 安装 Python 和 Pip (后端开发)
```bash
sudo apt install python3 python3-pip python3-venv -y
```
#### 1.3 安装 Node.js 和 npm (前端开发)
```bash
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt install nodejs -y
```
#### 1.4 安装 MySQL 数据库
```bash
sudo apt install mysql-server -y
sudo mysql_secure_installation
```
#### 1.5 安装 VSCode 编辑器（可选）
```bash
sudo apt install software-properties-common apt-transport-https wget -y
wget -q https://packages.microsoft.com/keys/microsoft.asc -O- | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://packages.microsoft.com/repos/vscode stable main"
sudo apt update
sudo apt install code -y
```
### 2. 设置 MySQL 数据库
#### 2.1 登录 MySQL
```bash
sudo mysql -u root -p
```
#### 2.2 创建数据库和用户
```sql
CREATE DATABASE to_do_list CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'todo_user'@'localhost' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON to_do_list.* TO 'todo_user'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

<a id="step2"></a>
## 第二部分：后端 Django 开发
### 1. 创建 Django 项目
#### 1.1 创建后端文件夹
```bash
mkdir to_do_list_project
cd to_do_list_project
mkdir backend
```
#### 1.2 创建 Python 虚拟环境并激活
```bash
python3 -m venv env
source env/bin/activate  # 在 Linux 环境下
```
#### 1.3 安装 Django 和其他依赖
```bash
pip install django djangorestframework mysqlclient
```
#### 1.4 初始化 Django 项目
```bash
cd backend
django-admin startproject backend .
python manage.py startapp tasks
```
#### 1.5 在 `settings.py` 中注册 app
打开 `backend/settings.py`，找到 `INSTALLED_APPS`，添加：
```python
INSTALLED_APPS += [
    'rest_framework',
    'tasks',
]
```
### 2. 配置 MySQL 数据库
在 `backend/settings.py` 中替换数据库配置：
```python
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': 'to_do_list',
        'USER': 'todo_user',
        'PASSWORD': 'password',
        'HOST': 'localhost',
        'PORT': '3306',
    }
}
```
测试数据库连接：
```bash
python manage.py migrate
```
### 3. 创建 `Task` 模型
编辑 `tasks/models.py`：
```python
from django.db import models

class Task(models.Model):
    title = models.CharField(max_length=255)
    done = models.BooleanField(default=False)
    created_at = models.DateTimeField(auto_now_add=True)

    def __str__(self):
        return self.title
```
运行以下命令生成数据库表：
```bash
python manage.py makemigrations
python manage.py migrate
```
### 4. 创建 RESTful API
#### 4.1 创建序列化器：`tasks/serializers.py`
```python
from rest_framework import serializers
from .models import Task

class TaskSerializer(serializers.ModelSerializer):
    class Meta:
        model = Task
        fields = '__all__'
```
#### 4.2 创建视图：`tasks/views.py`
```python
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status
from .models import Task
from .serializers import TaskSerializer

class TaskView(APIView):
    def get(self, request):
        tasks = Task.objects.all().order_by('-created_at')
        serializer = TaskSerializer(tasks, many=True)
        return Response(serializer.data)

    def post(self, request):
        serializer = TaskSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
```
#### 4.3 配置 URL 路由：`tasks/urls.py`
```python
from django.urls import path
from .views import TaskView

urlpatterns = [
    path('tasks/', TaskView.as_view(), name='tasks'),
]
```
#### 4.4 在 `backend/urls.py` 中包含子路由
```python
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/', include('tasks.urls')),
]
```
#### 4.5 启动开发服务器
```bash
python manage.py runserver
```
**测试后端 API**：
- GET `/api/tasks/` - 获取任务列表。
- POST `/api/tasks/` - 添加新任务。

<a id="step3"></a>
## 第三部分：前端 Vue.js 开发
### 1. 开发环境搭建
#### 1.1 安装 Vue CLI
如果尚未安装 Vue CLI，全局安装：
```bash
npm install -g @vue/cli
```
#### 1.2 创建 Vue 项目
进入 `to_do_list_project` 文件夹，并创建一个 Vue 项目：
```bash
cd to_do_list_project
vue create frontend
```
选择手动配置（Manually select features）并勾选以下选项：
- `Vue Router`（用于路由管理）
- `CSS Pre-processors`（如 SCSS）
然后完成项目创建，进入项目目录：
```bash
cd frontend
```
#### 1.3 安装依赖（Axios 和 ElementPlus）
```bash
npm install axios element-plus
```
### 2. 创建任务管理页面
#### 2.1 配置 Axios 全局设置
在 `src/plugins/axios.js` 中配置 Axios 用于后端通信：
```javascript
import axios from "axios";

const instance = axios.create({
  baseURL: "http://127.0.0.1:8000/api/", // Django 后端 API
  timeout: 5000,
  headers: {
    "Content-Type": "application/json",
  },
});

export default instance;
```
在主入口 `main.js` 中引入 Axios 和 Element Plus：
```javascript
import { createApp } from "vue";
import App from "./App.vue";
import axios from './plugins/axios';
import ElementPlus from 'element-plus'
import 'element-plus/dist/index.css';

const app = createApp(App);
app.config.globalProperties.$axios = axios;
app.use(ElementPlus);
app.mount("#app");
```
#### 2.2 创建 Task 组件
在 `src/components/TaskManager.vue` 中创建任务页面组件：
```vue
<template>
  <div class="task-manager">
    <el-container>
      <!-- 任务标题 -->
      <el-header>
        <h1>任务管理系统</h1>
      </el-header>

      <!-- 任务表单 -->
      <el-main>
        <el-form @submit.prevent="addTask" inline>
          <el-form-item>
            <el-input v-model="newTask" placeholder="添加新任务..."></el-input>
          </el-form-item>
          <el-form-item>
            <el-button type="primary" @click="addTask">添加任务</el-button>
          </el-form-item>
        </el-form>

        <!-- 任务列表 -->
        <el-table :data="tasks" style="width: 100%" v-if="tasks.length">
          <el-table-column prop="title" label="任务" />
          <el-table-column prop="done" label="状态">
            <template #default="{ row }">
              <el-switch
                v-model="row.done"
                active-text="完成"
                inactive-text="未完成"
                @change="updateTaskStatus(row)"
              />
            </template>
          </el-table-column>
          <el-table-column label="操作">
            <template #default="{ row }">
              <el-button type="danger" @click="deleteTask(row.id)">删除</el-button>
            </template>
          </el-table-column>
        </el-table>
      </el-main>
    </el-container>
  </div>
</template>

<script>
export default {
  data() {
    return {
      tasks: [], // 任务列表
      newTask: "", // 新任务内容
    };
  },
  methods: {
    // 获取任务列表
    fetchTasks() {
      this.$axios.get("tasks/").then((response) => {
        this.tasks = response.data;
      });
    },
    // 添加任务
    addTask() {
      if (!this.newTask.trim()) return;
      this.$axios
        .post("tasks/", { title: this.newTask })
        .then((response) => {
          this.tasks.unshift(response.data); // 添加到任务列表
          this.newTask = ""; // 清空输入框
        });
    },
    // 更新任务状态
    updateTaskStatus(task) {
      this.$axios
        .put(`tasks/${task.id}/`, { done: task.done })
        .catch((error) => {
          console.log(error);
          this.fetchTasks(); // 如果失败重新加载任务
        });
    },
    // 删除任务
    deleteTask(taskId) {
      this.$axios.delete(`tasks/${taskId}/`).then(() => {
        this.tasks = this.tasks.filter((task) => task.id !== taskId);
      });
    },
  },
  mounted() {
    // 页面加载时获取任务列表
    this.fetchTasks();
  },
};
</script>

<style>
.task-manager {
  width: 800px;
  margin: 20px auto;
}
h1 {
  margin: 0;
  padding: 15px;
  font-size: 24px;
  text-align: center;
}
</style>
```
#### 2.3 配置 Vue 路由
在 `src/router/index.js` 中设置路由：
```javascript
import { createRouter, createWebHistory } from "vue-router";
import TaskManager from "../components/TaskManager.vue";

const routes = [
  { path: "/", name: "Home", component: TaskManager },
];

const router = createRouter({
  history: createWebHistory(),
  routes,
});

export default router;
```
改造 `main.js` 以挂载路由：
```javascript
import router from './router';
app.use(router);
```
#### 2.4 启动前端开发服务器
```bash
npm run serve
```
访问 `http://localhost:8080` 测试前端功能。

<a id="step4"></a>
## 第四部分：部署项目
### 1. 准备 Docker 和 Docker Compose
确保你的服务器上已经安装了 Docker 和 Docker Compose。如果未安装，可以按照以下步骤进行安装：
#### 1.1 安装 Docker
```bash
sudo apt update
sudo apt install apt-transport-https ca-certificates curl software-properties-common -y
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
sudo apt update
sudo apt install docker-ce -y
sudo usermod -aG docker $USER
```
#### 1.2 安装 Docker Compose
```bash
sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```
### 2. 创建 Dockerfile
#### 2.1 后端 Dockerfile（`Dockerfile.backend`）
```Dockerfile
# 使用 Python 基础镜像
FROM python:3.9-slim

# 设置工作目录
WORKDIR /app

# 复制依赖文件
COPY backend/requirements.txt .

# 安装依赖
RUN pip install --no-cache-dir -r requirements.txt

# 复制项目文件
COPY backend .

# 暴露端口
EXPOSE 8000

# 启动后端服务
CMD ["gunicorn", "backend.wsgi:application", "--bind", "0.0.0.0:8000"]
```
#### 2.2 前端 Dockerfile（`Dockerfile.frontend`）
```Dockerfile
# 使用 Node.js 基础镜像
FROM node:18-alpine

# 设置工作目录
WORKDIR /app

# 复制依赖文件
COPY frontend/package*.json .

# 安装依赖
RUN npm install

# 复制项目文件
COPY frontend .

# 构建前端项目
RUN npm run build

# 使用 Nginx 作为静态文件服务器
FROM nginx:1.21-alpine

# 复制构建后的文件到 Nginx 目录
COPY --from=0 /app/dist /usr/share/nginx/html

# 暴露端口
EXPOSE 80

# 启动 Nginx
CMD ["nginx", "-g", "daemon off;"]
```
### 3. 创建 Docker Compose 文件
在项目根目录下创建 `docker-compose.yml` 文件：
```yaml
version: '3'

services:
  db:
    image: mysql:8.0
    environment:
      MYSQL_DATABASE: to_do_list
      MYSQL_USER: todo_user
      MYSQL_PASSWORD: password
      MYSQL_ROOT_PASSWORD: rootpassword
    volumes:
      - mysql-data:/var/lib/mysql
    ports:
      - "3306:3306"

  backend:
    build:
      context: .
      dockerfile: Dockerfile.backend
    environment:
      DATABASE_URL: mysql://todo_user:password@db:3306/to_do_list
    ports:
      - "8000:8000"
    depends_on:
      - db

  frontend:
    build:
      context: .
      dockerfile: Dockerfile.frontend
    ports:
      - "80:80"
    depends_on:
      - backend

volumes:
  mysql-data:
```
### 4. 构建和启动容器
在项目根目录下执行以下命令来构建和启动容器：
```bash
docker-compose build
docker-compose up -d
```
### 5. 验证部署
- **后端 API**：访问 `http://localhost:8000/api/tasks/`，应该能看到任务列表。
- **前端界面**：访问 `http://localhost`，应该能看到任务管理系统的界面。
### 6. 停止和清理容器
如果需要停止和清理容器，可以执行以下命令：
```bash
docker-compose down
```
---
## 补充
### 1. `requirements.txt`
这个文件包含了项目后端所需的 Python 依赖项。

```plaintext
django
djangorestframework
mysqlclient
gunicorn
```

### 2. `package.json`
这个文件包含了项目前端所需的 JavaScript 依赖项，以及一些脚本命令。

```json
{
  "name": "to_do_list_frontend",
  "version": "1.0.0",
  "description": "To-do list frontend using Vue.js",
  "scripts": {
    "serve": "vue-cli-service serve",
    "build": "vue-cli-service build"
  },
  "dependencies": {
    "axios": "^1.4.0",
    "element-plus": "^2.3.7",
    "vue": "^3.2.47",
    "vue-router": "^4.1.6"
  },
  "devDependencies": {
    "@vue/cli-service": "^5.0.8"
  },
  "browserslist": [
    "> 1%",
    "last 2 versions",
    "not dead"
  ]
}
```

### 3. `package-lock.json`
`package-lock.json` 是由 `npm` 自动生成的，用于锁定项目中所有依赖项的精确版本，确保每次安装时都能得到相同的依赖版本。由于它的内容通常很长，且会根据具体的安装环境和版本有所不同，这里提供一个生成它的步骤，你可以在项目的前端目录（`frontend`）下运行以下命令来生成：

```bash
cd frontend
npm install
```

运行上述命令后，`npm` 会根据 `package.json` 中的依赖项安装相应的包，并生成 `package-lock.json` 文件。

### 总结
- **`requirements.txt`**: 包含了后端 Python 依赖，使用 `pip install -r requirements.txt` 来安装。
- **`package.json`**: 包含了前端 JavaScript 依赖和脚本命令，使用 `npm install` 来安装依赖。
- **`package-lock.json`**: 由 `npm install` 自动生成，用于锁定依赖版本。

## 全部完成！
访问你配置的域名即可欣赏运行的任务管理系统。🎉
