# Dockerfile-backend

# Imagem base
FROM python:3.13

# Instalar o Astral UV
COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /bin/

# Assegurar o PATH
ENV PATH="/root/.local/bin/:$PATH"

# Expor a porta 8000
EXPOSE 8000

# Copiar os arquivos do projeto
COPY . /app

# Definir o diretório de trabalho no container
WORKDIR /app

# Instalar as dependências do projeto
RUN uv sync --frozen

# Comando para executar a aplicação
CMD ["uv", "run", "uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]