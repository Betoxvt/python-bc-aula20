# Aula 19 - Bootcamp Python | Jornada de Dados

Agora será o crud completo e com frontend.

- Backend: FastAPI
- Frontend: Com o Streamlit
- Banco de dados: PostgreSQL
- Containers: Docker, orquestrados pelo compose.yaml

## Backend

### database.py

Esse script configura a conexão e os recursos básicos para interagir com um banco de dados PostgreSQL usando SQLAlchemy.

#### 1. Configuração da URL do Banco de Dados

```python
SQLALCHEMY_DATABASE_URL = "postgresql://user:password@postgres/mydatabase"
```

**Propósito**: Define a URL de conexão com o banco de dados PostgreSQL.

**Formato**:

- postgresql://: Especifica o driver do banco de dados.
- user:password: Nome de usuário e senha para autenticação.
- @postgres: Endereço do servidor do banco de dados.
- /mydatabase: Nome do banco de dados ao qual vamos nos conectar.

#### 2. Criação do Motor de Conexão

```python
engine = create_engine(SQLALCHEMY_DATABASE_URL)
```

**Propósito**: Cria o "motor" (engine), que gerencia a conexão com o banco de dados.

**Como funciona**:

- O engine é o componente principal que conecta a aplicação ao banco de dados.
- Ele é responsável por traduzir as operações feitas pelo SQLAlchemy em comandos SQL para o banco de dados.

#### 3. Configuração da Sessão de Banco de Dados

```python
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)
```

**Propósito**: Configura sessões para interagir com o banco.

**Detalhes**:

- Uma sessão é um objeto que permite executar consultas (SELECT, INSERT, etc.) e manipular dados no banco.

- Opções:
    - autocommit=False: Desativa o commit automático, garantindo que você precise confirmar (commit) manualmente as transações.
    - autoflush=False: Desativa a escrita automática no banco antes de executar consultas. Isso dá mais controle, mas você precisa salvar explicitamente os dados com session.flush() ou session.commit().
    - bind=engine: Conecta a sessão ao motor de banco de dados configurado.

#### 4. Base para Modelos Declarativos

```python
Base = declarative_base()
```

**Propósito**: Cria uma classe base para os modelos ORM.

**Como funciona**:
- Todos os modelos (classes que representam tabelas no banco) devem herdar dessa Base.
- Essa base fornece ao SQLAlchemy informações necessárias para criar tabelas no banco de dados e mapear os dados para objetos Python.

#### 5. Gerenciador de Contexto para Sessões

```python
def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()
```

**Propósito**: Cria uma função utilitária para gerenciar sessões de banco de dados.

**Como funciona**:

- `db = SessionLocal()`: Abre uma nova sessão.
- `yield db`: Permite que o código que chama essa função utilize a sessão.
- `finally: db.close()`: Fecha a sessão automaticamente após o uso, evitando conexões abertas desnecessariamente.

Esse padrão é muito usado em frameworks como FastAPI para gerenciar dependências de banco de dados, garantindo que as sessões sejam abertas e fechadas corretamente.

**Resumindo**

1. Configura a conexão com o banco de dados PostgreSQL.
2. Prepara uma base para criar e mapear tabelas para objetos Python.
3. Fornece uma maneira fácil e segura de abrir e fechar sessões de banco para realizar consultas e transações.

---

### models.py

Esse script define um modelo de banco de dados usando SQLAlchemy. O modelo representa uma tabela chamada `products` e descreve como seus campos (colunas) se mapeiam para atributos de objetos Python.

#### 1. Importações:

- `Column`: Define colunas no banco de dados.
- **Tipos de dados** (`Integer`, `String`, `Float`, `DateTime`): Especificam os tipos de dados das colunas no banco.
- `func`: Permite usar funções SQL, como `NOW()` para obter a data/hora atual.
- `Base`: Importado do módulo `database`, é a classe base herdada para criar o modelo. Ela foi configurada no script `database.py`.

#### 2. Definição da Classe ProductModel

```python
class ProductModel(Base):
    __tablename__ = "products"  # esse será o nome da tabela
```

**Propósito**: Define um modelo ORM que representa a tabela products no banco de dados.

**Herdando de Base**:

- Informa ao SQLAlchemy que essa classe será mapeada para uma tabela no banco.
- A tabela será automaticamente registrada no esquema do banco ao criar as tabelas.

#### 3. Definição das Colunas

```python
    id = Column(Integer, primary_key=True) # Define que esta coluna é a chave primária da tabela.
    name = Column(String)
    description = Column(String)
    price = Column(Float)
    categoria = Column(String)
    email_fornecedor = Column(String)
    created_at = Column(DateTime(timezone=True), default=func.now()) # Usa a função SQL NOW() para definir a data/hora atual automaticamente ao criar um registro. Com suporte a fuso horário.
```

---

### schemas.py

#### 1. Importações
- `BaseModel`: Base para criar modelos de dados.
- **Tipos e validações**:
    - `PositiveFloat`: Garante que o preço seja positivo.
    - `EmailStr`: Valida e-mails.
    - `Optional`: Permite valores opcionais.
- **Outros**:
    - `Enum`: Define opções fixas para categorias.
    - `datetime`: Representa data/hora.
    - `field_validator`: Implementa validações customizadas.

#### 2. Enumeração de Categorias

```python
class CategoriaBase(Enum):
    ...
```

Define categorias fixas para produtos, garantindo consistência ao permitir apenas valores predefinidos.

#### 3. Modelos Pydantic

```python
class ProductBase(BaseModel):
    ...

    @field_validator("categoria")
    def check_categoria(cls, v):
        ...
```

- Valida campos básicos de produtos.
- A validação da categoria verifica se o valor está na lista de categorias definidas pelo Enum.

```python
class ProductCreate(ProductBase):
    pass
```

- Herda de ProductBase, validando os mesmos campos.
- Usado para criar novos produtos.

```python
class ProductResponse(ProductBase):
    ...

    class Config:
        from_attributes = True
```

- Configura a conversão automática de atributos de objetos ORM.

```python
class ProductUpdate(BaseModel):
    name: Optional[str] = None
    ...

    @field_validator("categoria", pre=True, always=True)
    def check_categoria(cls, v):
        ...
```

- Todos os campos são opcionais, permitindo atualizações parciais.
- Validação da categoria considera valores None.

**Resumindo**

- Organização: Estrutura modelos para operações com produtos (criação, leitura e atualização).
- Validação: Garante que os dados enviados e recebidos estejam corretos, com validações específicas.

---

### crud.py

Este módulo implementa as operações básicas de CRUD (Create, Read, Update, Delete) para gerenciar produtos utilizando SQLAlchemy.

**Funções Disponíveis**

Essas funções recebem uma instância do banco (Session) e utilizam os esquemas e modelos definidos no projeto para validar e persistir os dados.

#### 1. `get_product`

- **Propósito**: Recuperar um único produto pelo id.
- **Como funciona**:
    - Faz uma consulta no banco (`db.query`) buscando um registro na tabela `ProductModel` onde o campo `id` corresponde ao `product_id` informado.
    - Retorna o primeiro resultado encontrado ou `None` caso o `id` não exista.
- **Destaque**: Ideal para buscar detalhes de um produto específico.

#### 2. `get_products`

- **Propósito**: Recuperar todos os produtos do banco de dados.
- **Como funciona**:
    - Faz uma consulta completa na tabela `ProductModel` usando `.all()`, retornando uma lista com todos os registros.
- **Destaque**: Útil para exibir um catálogo completo ou listar produtos em uma API.

#### 3. `create_product`

- **Propósito**: Inserir um novo produto no banco de dados.
- **Como funciona**:
    1. Recebe um objeto do tipo `ProductCreate` com os dados do produto.
    2. Usa `**product.model_dump()` para transformar os dados do esquema em argumentos para o modelo ORM.
    3. Adiciona o novo objeto (`db.add`) à sessão do banco e confirma a transação com db.commit.
    4. Usa `db.refresh` para sincronizar o objeto criado com os dados gerados pelo banco (como o `id`).
    5. Retorna o produto criado.
- **Destaque**: Automação completa para criar novos registros, incluindo validação de dados pelo esquema.

#### 4. `delete_product`

- **Propósito**: Remover um produto pelo `id`.
- **Como funciona**:
    1. Busca o produto no banco com o `product_id` usando `db.query`.
    2. Se o produto for encontrado, o remove com `db.delete` e confirma a transação com `db.commit`.
    3. Retorna o produto removido ou `None` se o `id` não existir.
- **Destaque**: Remove o produto do banco, garantindo que as mudanças sejam permanentes após o `commit`.

#### 5. `update_product`

- **Propósito**: Atualizar os dados de um produto existente.
- **Como funciona**:
    1. Busca o produto no banco pelo `id` usando `db.query`.
    2. Se o produto não for encontrado, retorna `None`.
    3. Atualiza apenas os campos fornecidos no objeto `ProductUpdate` (campos opcionais).
    4. Salva as mudanças com `db.commit` e atualiza o objeto em memória com `db.refresh`.
    5. Retorna o produto atualizado.
- **Destaque**: Suporta atualizações parciais, permitindo modificar apenas os campos necessários.

---

### router.py

Este script define rotas RESTful para gerenciar produtos utilizando FastAPI. As rotas interagem com o banco de dados por meio das funções CRUD previamente implementadas, retornando respostas padronizadas com validação de dados.

#### **1. Rota: Criar Produto**
```python
@router.post("/products/", response_model=ProductResponse)
def create_product_route(product: ProductCreate, db: Session = Depends(get_db)):
    return create_product(db=db, product=product)
```
- **Método HTTP**: `POST`
- **URL**: `/products/`
- **Entrada**:
  - Objeto `ProductCreate` (nome, descrição, preço, categoria, e-mail do fornecedor).
- **Saída**: Retorna o produto criado no formato do esquema `ProductResponse`.
- **Descrição**: Insere um novo produto no banco de dados.

#### **2. Rota: Listar Todos os Produtos**
```python
@router.get("/products/", response_model=List[ProductResponse])
def read_all_products_route(db: Session = Depends(get_db)):
    products = get_products(db)
    return products
```
- **Método HTTP**: `GET`
- **URL**: `/products/`
- **Entrada**: Nenhuma.
- **Saída**: Retorna uma lista de todos os produtos cadastrados.
- **Descrição**: Recupera todos os registros da tabela de produtos.

#### **3. Rota: Buscar Produto pelo ID**
```python
@router.get("/products/{product_id}", response_model=ProductResponse)
def read_product_route(product_id: int, db: Session = Depends(get_db)):
    db_product = get_product(db, product_id=product_id)
    if db_product is None:
        raise HTTPException(status_code=404, detail="Product not found")
    return db_product
```
- **Método HTTP**: `GET`
- **URL**: `/products/{product_id}`
- **Entrada**:
  - `product_id`: ID do produto a ser buscado.
- **Saída**: Retorna o produto correspondente ou um erro 404 se não encontrado.
- **Descrição**: Busca e retorna os detalhes de um produto específico.

#### **4. Rota: Deletar Produto**
```python
@router.delete("/products/{product_id}", response_model=ProductResponse)
def detele_product_route(product_id: int, db: Session = Depends(get_db)):
    db_product = delete_product(db, product_id=product_id)
    if db_product is None:
        raise HTTPException(status_code=404, detail="Product not found")
    return db_product
```
- **Método HTTP**: `DELETE`
- **URL**: `/products/{product_id}`
- **Entrada**:
  - `product_id`: ID do produto a ser deletado.
- **Saída**: Retorna o produto removido ou um erro 404 se não encontrado.
- **Descrição**: Remove o produto identificado pelo `id` do banco de dados.

#### **5. Rota: Atualizar Produto**
```python
@router.put("/products/{product_id}", response_model=ProductResponse)
def update_product_route(
    product_id: int, product: ProductUpdate, db: Session = Depends(get_db)
):
    db_product = update_product(db, product_id=product_id, product=product)
    if db_product is None:
        raise HTTPException(status_code=404, detail="Product not found")
    return db_product
```
- **Método HTTP**: `PUT`
- **URL**: `/products/{product_id}`
- **Entrada**:
  - `product_id`: ID do produto a ser atualizado.
  - Objeto `ProductUpdate` com os campos a serem modificados.
- **Saída**: Retorna o produto atualizado ou um erro 404 se não encontrado.
- **Descrição**: Atualiza as informações de um produto existente.

#### **Resumindo**
1. **Organização**: As rotas estão encapsuladas em `router.py` para modularidade e reutilização.
2. **Integração CRUD**: Cada rota utiliza uma função CRUD para interagir com o banco de dados.
3. **Validação**: Os dados de entrada e saída são validados pelos esquemas Pydantic.
4. **Erros HTTP**: Lança exceções apropriadas (ex.: 404 para produto não encontrado).
5. **API RESTful**: As rotas seguem as convenções REST, facilitando o uso em integrações ou aplicativos frontend.

---

### main.py

Este script é o ponto de entrada principal da aplicação em FastAPI. Ele inicializa o framework, configura o banco de dados e registra as rotas.

#### 1. **Importações**
- **FastAPI**: Framework utilizado para construir a API.
- **engine**: Instância do mecanismo de banco de dados criada em `database.py`.
- **models**: Modelos declarativos do SQLAlchemy definidos em `models.py`.
- **router**: Conjunto de rotas definidas em `router.py`.

#### 2. **Criação do Banco de Dados**
```python
models.Base.metadata.create_all(bind=engine)
```
- **Descrição**: Gera as tabelas no banco de dados com base nos modelos definidos, caso elas ainda não existam.

#### 3. **Inicialização do FastAPI**
```python
app = FastAPI()
```
- **Descrição**: Cria uma instância da aplicação FastAPI para gerenciar as requisições.

#### 4. **Registro de Rotas**
```python
app.include_router(router)
```
- **Descrição**: Inclui o conjunto de rotas definidas em `router.py` na aplicação.

#### **Resumindo**
- **Inicializa o framework FastAPI.**
- **Cria o esquema do banco de dados se necessário.**
- **Configura as rotas da aplicação para interação com os endpoints.**