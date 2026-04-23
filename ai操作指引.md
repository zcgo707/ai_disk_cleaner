好的，我将详细介绍如何使用 Docker Compose 部署 vLLM 和 Milvus。

首先，我们需要联网搜索最新的部署指南和配置，确保使用最佳实践。看起来网络搜索遇到了一些问题，无法获取最新信息。不过没关系，我可以根据通用知识为你提供详细的部署指南。

以下是在 WSL 环境下使用 Docker Compose 部署 vLLM 和 Milvus 的完整步骤。

### 第一步：准备工作

1.  **确认 WSL2 和 Docker 环境**：确保你已经在 WSL2 中安装了 Docker 和 Docker Compose，并且能够访问 Windows 上的 Docker Desktop（或 WSL2 中的 Docker 引擎）。你可以通过 `docker --version` 和 `docker compose version` 验证。
2.  **GPU 支持**（可选，但 vLLM 通常需要）：
    *   确保你的 Windows 主机已安装支持 CUDA 的 NVIDIA 驱动。
    *   在 WSL2 中安装 `nvidia-container-toolkit`。参考 NVIDIA 官方文档进行安装和配置。这一步是为了让 Docker 容器能够访问 GPU。
    *   通过 `nvidia-smi` 在 WSL2 终端中确认 GPU 可用。

### 第二步：部署 Milvus (Standalone)

Milvus 依赖于 MinIO/Etcd 存储和元数据服务。Docker Compose 是部署这种多组件服务的最佳方式。

1.  **创建 Milvus 配置目录**：
    ```bash
    mkdir ~/milvus_standalone && cd ~/milvus_standalone
    ```
2.  **下载官方提供的 `docker-compose.yml` 文件**：
    你可以从 Milvus GitHub 发布页面获取最新的配置文件。这里以一个常见的版本为例，你需要去 [Milvus Releases](https://github.com/milvus-io/milvus/releases) 页面查找最新的 `milvus-standalone-docker-compose.yml` 文件链接。
    ```bash
    # 请访问 Milvus Releases 页面，找到最新的 standalone compose 文件 URL 替换下面的 URL
    # 例如，假设最新的版本是 v2.6.0 (请注意，版本号可能会变)
    wget https://github.com/milvus-io/milvus/releases/latest/download/milvus-standalone-docker-compose.yml -O docker-compose.yml
    ```
    *   **注意**：`latest` 链接可能指向不稳定版本，建议明确指定版本号，如 `v2.6.0`。
    *   **或者**，你也可以手动创建 `docker-compose.yml` 文件，内容如下（这个是基于常见配置简化而来，具体请以官方为准）：
        ```yaml
        # docker-compose.yml for Milvus standalone
        version: '3.5'

        services:
          etcd:
            container_name: milvus-etcd
            image: quay.io/coreos/etcd:v3.5.5
            environment:
              - ETCD_AUTO_COMPACTION_MODE=revision
              - ETCD_AUTO_COMPACTION_RETENTION=1000
              - ETCD_QUOTA_BACKEND_BYTES=4294967296
              - ETCD_SNAPSHOT_COUNT=50000
            volumes:
              - ${PWD}/volumes/etcd:/etcd-data
            command: etcd -data-dir=/etcd-data -advertise-client-urls=http://127.0.0.1:2379 -listen-client-urls http://0.0.0.0:2379 -listen-peer-urls http://0.0.0.0:2380 -name=etcd -initial-advertise-peer-urls http://127.0.0.1:2380 -initial-cluster=etcd=http://127.0.0.1:2380 -initial-cluster-state=new -initial-cluster-token=tkn

          minio:
            container_name: milvus-minio
            image: minio/minio:RELEASE.2023-09-04T19-57-37Z
            environment:
              MINIO_ROOT_USER: minioadmin
              MINIO_ROOT_PASSWORD: minioadmin
            volumes:
              - ${PWD}/volumes/minio:/minio_data
            command: minio server /minio_data --console-address ":9001"
            healthcheck:
              test: ["CMD", "curl", "-f", "http://localhost:9000/minio/health/live"]
              interval: 30s
              timeout: 20s
              retries: 3

          standalone:
            container_name: milvus-standalone
            image: milvusdb/milvus:v2.6.0 # 请替换为你要使用的版本
            command: ["milvus", "run", "standalone"]
            environment:
              - ETCD_ENDPOINTS=etcd:2379
              - MINIO_ADDRESS=minio:9000
            volumes:
              - ${PWD}/volumes/milvus:/var/lib/milvus
            ports:
              - "19530:19530"
              - "9091:9091" # Milvus UI
            depends_on:
              - etcd
              - minio
            healthcheck:
              test: ["CMD", "curl", "-f", "http://localhost:9091/state"] # Check UI port
              interval: 30s
              timeout: 20s
              retries: 3

          # Optional: Attu UI for Milvus
          attu:
             container_name: milvus-attu
             image: zilliz/attu:latest
             ports:
               - "8000:3000"
             environment:
               MILVUS_URL: standalone:19530 # Use service name from this compose file
             depends_on:
               - standalone
             links:
               - standalone:standalone
        ```
3.  **启动 Milvus 服务**：
    ```bash
    cd ~/milvus_standalone
    docker compose up -d
    ```
    *   `-d` 参数表示在后台运行。
4.  **检查服务状态**：
    ```bash
    docker compose ps
    ```
    你应该能看到 `milvus-etcd`, `milvus-minio`, `milvus-standalone`, (可能还有 `milvus-attu`) 都处于 `running` 状态。
5.  **等待初始化完成**：Milvus 初始化可能需要几分钟。你可以通过 `docker logs milvus-standalone` 查看日志，直到看到类似 "Milvus Proxy successfully started" 的信息。
6.  **访问 Milvus**：
    *   服务端口：`19530` (gRPC), `9091` (UI)
    *   UI 地址：在 WSL 浏览器中访问 `http://localhost:9091` 或 `http://localhost:8000` (如果启动了 Attu)。

### 第三步：部署 vLLM

vLLM 需要一个单独的 Docker 镜像来运行。我们将创建一个自定义的 Dockerfile 来构建 vLLM 服务镜像，并将其添加到与 Milvus 相同的 `docker-compose.yml` 中。

1.  **创建 vLLM 项目目录**：
    ```bash
    mkdir ~/vllm_service && cd ~/vllm_service
    ```
2.  **创建 vLLM 的 Dockerfile**：
    ```dockerfile
    # Dockerfile
    FROM nvidia/cuda:12.1-devel-ubuntu20.04 # 选择匹配你 GPU 驱动的 CUDA 基础镜像

    # 安装 Python 3.10 和 pip
    RUN apt-get update && apt-get install -y --no-install-recommends \
        python3.10 \
        python3-pip \
        python3.10-venv \
        && rm -rf /var/lib/apt/lists/*

    # 升级 pip
    RUN pip3 install --upgrade pip

    # 安装 vLLM (以及可能需要的其他库)
    # 注意：确保 CUDA 版本与基础镜像匹配
    RUN pip3 install vllm

    # 设置工作目录
    WORKDIR /app

    # 默认启动命令，可以通过 docker run 或 docker compose 覆盖
    # CMD ["python", "-m", "vllm.entrypoints.openai.api_server", "--help"]
    ```
3.  **修改 Milvus 的 `docker-compose.yml`**（位于 `~/milvus_standalone/docker-compose.yml`）以添加 vLLM 服务：
    ```yaml
    # docker-compose.yml (Updated to include vLLM)
    version: '3.5'

    services:
      etcd:
        container_name: milvus-etcd
        image: quay.io/coreos/etcd:v3.5.5
        environment:
          - ETCD_AUTO_COMPACTION_MODE=revision
          - ETCD_AUTO_COMPACTION_RETENTION=1000
          - ETCD_QUOTA_BACKEND_BYTES=4294967296
          - ETCD_SNAPSHOT_COUNT=50000
        volumes:
          - ${PWD}/volumes/etcd:/etcd-data
        command: etcd -data-dir=/etcd-data -advertise-client-urls=http://127.0.0.1:2379 -listen-client-urls http://0.0.0.0:2379 -listen-peer-urls http://0.0.0.0:2380 -name=etcd -initial-advertise-peer-urls http://127.0.0.1:2380 -initial-cluster=etcd=http://127.0.0.1:2380 -initial-cluster-state=new -initial-cluster-token=tkn

      minio:
        container_name: milvus-minio
        image: minio/minio:RELEASE.2023-09-04T19-57-37Z
        environment:
          MINIO_ROOT_USER: minioadmin
          MINIO_ROOT_PASSWORD: minioadmin
        volumes:
          - ${PWD}/volumes/minio:/minio_data
        command: minio server /minio_data --console-address ":9001"
        healthcheck:
          test: ["CMD", "curl", "-f", "http://localhost:9000/minio/health/live"]
          interval: 30s
          timeout: 20s
          retries: 3

      standalone:
        container_name: milvus-standalone
        image: milvusdb/milvus:v2.6.0 # 请替换为你要使用的版本
        command: ["milvus", "run", "standalone"]
        environment:
          - ETCD_ENDPOINTS=etcd:2379
          - MINIO_ADDRESS=minio:9000
        volumes:
          - ${PWD}/volumes/milvus:/var/lib/milvus
        ports:
          - "19530:19530"
          - "9091:9091"
        depends_on:
          - etcd
          - minio
        healthcheck:
          test: ["CMD", "curl", "-f", "http://localhost:9091/state"]
          interval: 30s
          timeout: 20s
          retries: 3

      # Optional: Attu UI for Milvus
      attu:
         container_name: milvus-attu
         image: zilliz/attu:latest
         ports:
           - "8000:3000"
         environment:
           MILVUS_URL: standalone:19530
         depends_on:
           - standalone
         links:
           - standalone:standalone

      # vLLM Service
      vllm-service:
        container_name: my_vllm_container
        build: # Build from the Dockerfile in the current directory (~/milvus_standalone)
          context: . # Path relative to where docker-compose.yml is located
          dockerfile: ../vllm_service/Dockerfile # Point to the Dockerfile you created
        ports:
          - "8000:8000" # Expose vLLM API port. Note: This conflicts with Attu's port 8000. Change one of them.
        volumes:
          # - /path/to/your/model/on/host:/app/model # Mount your model directory into the container
          - ./models:/app/model # Example: mount a 'models' dir inside ~/milvus_standalone
        # Connect to the same network as Milvus components
        networks:
          default: # Uses the default network created by docker-compose
        # Enable GPU access (WSL2 + nvidia-docker)
        deploy:
          resources:
            reservations:
              devices:
                - driver: nvidia
                  count: 1 # Or "all" if you want to use all GPUs
                  capabilities: [gpu]
        # Command to start vLLM API server
        # Replace "/app/model" with the path to your actual model inside the container
        command: >
          python -m vllm.entrypoints.openai.api_server
          --model /app/model # Use the mounted path
          --tensor-parallel-size 1 # Adjust based on your GPU count
          --host 0.0.0.0 # Listen on all interfaces within the container
          --port 8000

    # Define networks explicitly if needed (optional, default is created automatically)
    # networks:
    #   default:
    #     driver: bridge

    # Define volumes explicitly if needed (optional, anonymous volumes work too)
    # volumes:
    #   etcd_data:
    #   minio_data:
    #   milvus_data:
    ```
    *   **注意**：上面的例子中，vLLM 和 Attu 都映射到了主机的 `8000` 端口，这会产生冲突。你需要将其中一个服务的主机端口改掉，例如将 Attu 改为 `8001:3000`，或将 vLLM 改为 `8001:8000`。
    *   **注意**：`command` 中的 `--model` 参数需要指向容器内的模型路径。你需要将模型文件放在 WSL 的某个位置（例如 `~/milvus_standalone/models`），然后通过 `volumes` 部分挂载到容器内的 `/app/model` 路径。
    *   **注意**：`deploy.resources.reservations.devices` 部分用于请求 GPU 资源。确保 WSL2 和 Docker 已正确配置 GPU 支持。
4.  **放置模型**（如果需要）：
    在 `~/milvus_standalone` 目录下创建一个 `models` 文件夹（或你在 `volumes` 中指定的任何名称），并将你的 Hugging Face 模型文件夹复制进去。
    ```bash
    mkdir ~/milvus_standalone/models
    # cp -r /path/to/your/huggingface/model ~/milvus_standalone/models/
    ```
5.  **重新启动组合服务**：
    ```bash
    cd ~/milvus_standalone
    docker compose down # Stop existing services if any
    docker compose up -d # Build vLLM image (if changed) and start all services
    ```
6.  **检查所有服务状态**：
    ```bash
    docker compose ps
    ```
    你现在应该能看到 Milvus 相关的容器以及 `my_vllm_container` 都在运行。
7.  **访问服务**：
    *   **Milvus**: `http://localhost:19530` (API), `http://localhost:9091` (UI) 或 `http://localhost:8000` (Attu - if not conflicting)
    *   **vLLM**: `http://localhost:8000` (API) (or the port you chose if changed due to conflict)。你可以用 `curl` 或 OpenAI API 兼容的客户端来测试。

通过这种方式，你就在 WSL 环境中成功部署了一个独立的 Milvus 向量数据库和一个 vLLM 推理服务，并且它们可以通过内部网络相互通信。