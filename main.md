# Projeto: Sistema de Gerenciamento de Biblioteca no PostgreSQL

Esse projeto envolve a criação de um banco de dados para gerenciar uma biblioteca, incluindo livros, usuários, empréstimos e devoluções.

### Tecnologias:

✅ PostgreSQL rodando em Linux ou Windows

✅ Uso de schemas, views, funções, triggers e índices

✅ Controle de permissões e autenticação

## Passo 1: Criação do Banco de Dados

```PGSQL
CREATE DATABASE biblioteca;
```

## Acesso pelo terminal

```
psql -U postgres -d biblioteca
```

## Passo 2: Criação das tabelas

```PGSQL
-- Tabela de usuários
CREATE TABLE usuarios (
    id SERIAL PRIMARY KEY,
    nome VARCHAR(100) NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    telefone VARCHAR(15),
    criado_em TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Tabela de livros
CREATE TABLE livros (
    id SERIAL PRIMARY KEY,
    titulo VARCHAR(255) NOT NULL,
    autor VARCHAR(100) NOT NULL,
    ano_publicacao INT CHECK (ano_publicacao > 0),
    genero VARCHAR(50),
    disponivel BOOLEAN DEFAULT TRUE
);

-- Tabela de empréstimos
CREATE TABLE emprestimos (
    id SERIAL PRIMARY KEY,
    usuario_id INT REFERENCES usuarios(id) ON DELETE CASCADE,
    livro_id INT REFERENCES livros(id) ON DELETE CASCADE,
    data_emprestimo DATE DEFAULT CURRENT_DATE,
    data_devolucao DATE,
    status VARCHAR(20) CHECK (status IN ('emprestado', 'devolvido')) DEFAULT 'emprestado'
);
```

### Explicando as variáveis de criação

```PGSQL
criado_em TIMESTAMP DEFAULT CURRENT_TIMESTAMP
```

Será definida a data de criação do usuário no momento da criação do mesmo.









