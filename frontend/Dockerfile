# Dockerfile-frontend

# Imagem base
FROM python:3.13

# Instalar o Astral UV
COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /bin/

# Assegurar o PATH
ENV PATH="/root/.local/bin/:$PATH"

# Expor a porta 8501
EXPOSE 8501

# Copiar os arquivos do projeto
COPY . /app

# Definir o diretório de trabalho no container
WORKDIR /app

# Instalar as dependências do projeto
RUN uv pip sync requirements.txt

# Comando para executar a aplicação
CMD ["uv", "run", "streamlit", "run", "app.py", "--server.port=8501", "--server.address=0.0.0.0"]
