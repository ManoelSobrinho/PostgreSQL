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

```PGSQL
ano_publicacao INT CHECK (ano_publicacao > 0)
```

Impede que o ano de publicação do livro seja menor ou igual a 0.

```PGSQL
usuario_id INT REFERENCES usuarios(id) ON DELETE CASCADE
livro_id INT REFERENCES livros(id) ON DELETE CASCADE
```

Garantem que quando um usuário ou livro for excluído, todo registro referente a eles na tabela de empréstimos também será excluído.

## Passo 3: Criação de Índices para Melhorar Performance

Alguns índices que podem ajudar a performance das consultas nesse modelo de negócio:

```PGSQL
CREATE INDEX idx_usuarios_email ON usuarios(email);
CREATE INDEX idx_livros_titulo ON livros(titulo);
CREATE INDEX idx_emprestimos_status ON emprestimos(status);
```

### Explicação

Algumas buscas comuns serão melhoradas, nesse caso, buscas pelo e-mail do usuário que foi cadastrado, buscas pelo título do livro e também pelo status do empréstimo.

# Passo 4: Criação de Views para Consultas Rápidas

Podemos melhorar ainda mais a performance das consultas transformando uma consulta muito comum em uma VIEW, neste caso um exemplo seria a criação de uma VIEW para visualizar os empréstimos ativos.

```PGSQL
CREATE VIEW emprestimos_ativos AS
SELECT e.id, u.nome AS usuario, l.titulo AS livro, e.data_emprestimo 
FROM emprestimos e
JOIN usuarios u ON e.usuario_id = u.id
JOIN livros l ON e.livro_id = l.id
WHERE e.status = 'emprestado';
```

Podemos visualizar com a seguinte query:

```PGSQL
SELECT * FROM emprestimos_ativos;
```

# Passo 5: Criação de Função para Devolver Livros

Outra facilidade que poder ser criada é uma função para devolver livros que estão emprestados.

```PGSQL
CREATE FUNCTION devolver_livro(emprestimo_id INT) RETURNS VOID AS $$
BEGIN
    UPDATE emprestimos
    SET status = 'devolvido', data_devolucao = CURRENT_DATE
    WHERE id = emprestimo_id;
    
    UPDATE livros
    SET disponivel = TRUE
    WHERE id = (SELECT livro_id FROM emprestimos WHERE id = emprestimo_id);
END;
$$ LANGUAGE plpgsql;
```

Esta função irá fazer com que um livro que está emprestado seja devolvido e altere o status do mesmo tanto na tabela emprestimos como na tabela livros.

Exemplo: Devolução do empréstimo que tem o id = 1:

```PGSQL
SELECT devolver_livro(1);
```

# Passo 6: Criação de Trigger para Atualizar Disponibilidade

Podemos também criar uma trigger que atualize a disponibilidade do livro.

```PGSQL
CREATE OR REPLACE FUNCTION atualizar_disponibilidade() RETURNS TRIGGER AS $$
BEGIN
    IF NEW.status = 'emprestado' THEN
        UPDATE livros SET disponivel = FALSE WHERE id = NEW.livro_id;
    ELSIF NEW.status = 'devolvido' THEN
        UPDATE livros SET disponivel = TRUE WHERE id = NEW.livro_id;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_atualizar_disponibilidade
AFTER INSERT OR UPDATE ON emprestimos
FOR EACH ROW EXECUTE FUNCTION atualizar_disponibilidade();
```

# Passo 7: Criação de Função para emprestar livro

Podemos criar também uma função que facilite o empréstimo de um livro.

```PGSQL
CREATE OR REPLACE FUNCTION emprestar_livro(p_usuario_id INT, p_livro_id INT)
RETURNS TEXT AS $$
DECLARE
    v_disponivel BOOLEAN;
BEGIN
    -- Verificar se o livro está disponível
    SELECT disponivel INTO v_disponivel FROM livros WHERE id = p_livro_id;

    IF v_disponivel IS NULL THEN
        RETURN 'Erro: Livro não encontrado.';
    ELSIF v_disponivel = FALSE THEN
        RETURN 'Erro: Livro já está emprestado.';
    END IF;

    -- Inserir o empréstimo
    INSERT INTO emprestimos (usuario_id, livro_id)
    VALUES (p_usuario_id, p_livro_id);

    -- Atualizar o status do livro para indisponível
    UPDATE livros SET disponivel = FALSE WHERE id = p_livro_id;

    RETURN 'Livro emprestado com sucesso!';
END;
$$ LANGUAGE plpgsql;
```

# Passo 8: Criação de Usuários e Configuração de Permissões

Por fim podemos criar usuários e definir permissões para acesso ao banco de dados.

Exemplo: Criação de usuário leitor que pode visualizar todas as tabelas pertencentes ao SCHEMA public.

```PGSQL
CREATE USER leitor WITH PASSWORD 'senha123';
GRANT CONNECT ON DATABASE biblioteca TO leitor;
GRANT USAGE ON SCHEMA public TO leitor;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO leitor;
```

Exemplo: Criação de usuário bibliotecario que tem permissão para gerenciar os livros.

```PGSQL
CREATE USER bibliotecario WITH PASSWORD 'senha456';
GRANT INSERT, UPDATE, DELETE ON livros TO bibliotecario;
```

# Na prática

## Criando usuários e livros

Alguns usuários e livros foram adicionados para verificar exemplos práticos.

<p align="left">
<img src="https://github.com/user-attachments/assets/25ae9ffa-bbb5-4828-8959-099cdf970fc8">
</p>

<p align="left">
<img src="https://github.com/user-attachments/assets/8fa58c8d-8f5a-47ca-928f-62295572a8c8">
</p>

## Empréstimo de livro

### Emprestando um livro com id=1 ao usuário de id=1

```PGSQL
SELECT emprestar_livro(1, 1);
```

<p align="left">
<img src="https://github.com/user-attachments/assets/9d98f7ad-2144-4cf8-bc89-00efc114e060">
</p>

### Emprestando o livro de id=1 que já está emprestado a outro usuário

<p align="left">
<img src="https://github.com/user-attachments/assets/46d84d88-9784-437c-9eef-fff1eb6519e7">
</p>

Não irá funcionar pois o livro já está emprestado. É possível ver na tabela que o campo disponivel foi atualizado pela trigger.

<p align="left">
<img src="https://github.com/user-attachments/assets/b2fd1406-4472-4def-97cc-6dcc41022559">
</p>

## Devolução de um livro

Vamos devolver o livro de id=1 que está emprestado ao usuário de id=1

```PGSQL
SELECT devolver_livro(1)
```

Com isso, podemos ver que, ao consultar as tabelas livros e emprestimos os campos referentes ao status do livro forama alterados e o livro pode ser emprestado novamente.

<p align="left">
<img src="https://github.com/user-attachments/assets/a55a2f2e-88ee-4dd6-9a3b-2ef84deee875">
</p>

<p align="left">
<img src="https://github.com/user-attachments/assets/ac9a467d-8625-420a-90b5-e20ddeb98507">
</p>

## O que acontece se um livro que está emprestado ou um usuário que tem um empréstimo ativo são excluídos?

Como teste emprestei fiz os seguintes empréstimos:

Livro com id=2 foi emprestado ao usuário de id=2;

Livro com id=3 foi emprestado ao usuário de id=3;

Com os seguintes comandos:

```PGSQL
SELECT emprestar_livro(2, 2);
SELECT emprestar_livro(3, 3);
```

Notemos que o campo disponivel na tabela livros já foi alterado para false e duas novas linhas apareceram na tabela emprestimos

<p align="left">
<img src="https://github.com/user-attachments/assets/eb384ea8-9fe1-4405-b55e-baf500a21251">
</p>

<p align="left">
<img src="https://github.com/user-attachments/assets/d0c1586d-0961-47b6-b4a0-afe866b89cd9">
</p>










