# 🛒 Advanced E-Commerce Database Lab (MySQL)

Este repositório contém uma implementação limpa e de nível de produção de estruturas avançadas de bancos de dados em **MySQL**. Utilizando o cenário clássico de um e-commerce como ambiente de testes (*sandbox*), este projeto demonstra a aplicação prática de arquitetura relacional, rotinas automatizadas, isolamento de dados, simulação de cache de performance e gerenciamento granular de privilégios de acesso.

O script SQL está totalmente limpo, livre de comentários internos e otimizado para execução direta.

---

## 🏗️ Arquitetura e Fluxo de Dados

A arquitetura do projeto é dividida em três camadas operacionais distintas projetadas para simular o comportamento de sistemas corporativos reais:

1. **Camada Transacional (OLTP):** Gerencia as operações principais de escrita e persistência através das tabelas `produtos` e `vendas`.
2. **Camada de Automação e Auditoria:** Intercepta mutações de estado através de gatilhos nativos (`trg_auditoria_preco`) para alimentar o histórico da tabela `auditoria_precos` de forma assíncrona.
3. **Camada Analítica e de Segurança:** Restringe a exposição da infraestrutura base através de virtualização de dados (`vw_produtos_disponiveis`) e otimiza consultas de relatórios pesados através de um simulador de Views Materializadas baseado em eventos (`mv_resumo_vendas`).

[ Aplicação / Cliente ]
           │
           ├─── (Executa Escrita/DML) ──► [ Tabelas Transacionais ] ──► (Trigger) ──► [ Tabela de Auditoria ]
           │                                      │
           ├─── (Leitura Restrita) ─────► [ Views Virtualizadas ]
           │
           └─── (Consultas Analíticas) ─► [ Materialized Views (Cache) ]


---

## 📋 Código SQL de Implantação (Clean Code)

Copie e execute o bloco de script abaixo diretamente no terminal do seu cliente MySQL, MySQL Workbench ou pipeline de migração:

```sql
CREATE DATABASE lab_ecommerce;
USE lab_ecommerce;

CREATE TABLE produtos (
    id INT AUTO_INCREMENT PRIMARY KEY,
    nome VARCHAR(100),
    preco DECIMAL(10,2),
    estoque INT
);

CREATE TABLE vendas (
    id INT AUTO_INCREMENT PRIMARY KEY,
    produto_id INT,
    quantidade INT,
    total DECIMAL(10,2),
    data_venda DATETIME DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (produto_id) REFERENCES produtos(id)
);

CREATE TABLE auditoria_precos (
    id INT AUTO_INCREMENT PRIMARY KEY,
    produto_id INT,
    preco_antigo DECIMAL(10,2),
    preco_novo DECIMAL(10,2),
    data_alteracao DATETIME DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO produtos (nome, preco, estoque) VALUES
('Notebook', 3500.00, 10),
('Smartphone', 2000.00, 20),
('Teclado Mecânico', 350.00, 15);

CREATE VIEW vw_produtos_disponiveis AS
SELECT
    nome AS 'Produto',
    preco AS 'Valor',
    estoque AS 'Quantidade Disponivel'
FROM produtos
WHERE estoque > 0;

SELECT * FROM vw_produtos_disponiveis;

DELIMITER //
CREATE FUNCTION fn_calcular_desconto (valor DECIMAL(10,2), qtd INT)
RETURNS DECIMAL(10,2)
DETERMINISTIC
BEGIN
    DECLARE valor_final DECIMAL(10,2);
    IF qtd >= 3 THEN
        SET valor_final = (valor * qtd) * 0.90;
    ELSE
        SET valor_final = valor * qtd;
    END IF;
    RETURN valor_final;
END //
DELIMITER ;

SELECT fn_calcular_desconto(350.00, 4) AS Total_Com_Desconto;

DELIMITER //
CREATE TRIGGER trg_auditoria_preco
AFTER UPDATE ON produtos
FOR EACH ROW
BEGIN
    IF OLD.preco <> NEW.preco THEN
        INSERT INTO auditoria_precos (produto_id, preco_antigo, preco_novo)
        VALUES (OLD.id, OLD.preco, NEW.preco);
    END IF;
END //
DELIMITER ;

UPDATE produtos SET preco = 3200.00 WHERE id = 1;
SELECT * FROM auditoria_precos;

DELIMITER //
CREATE PROCEDURE sp_adicionar_estoque (IN p_produto_id INT, IN p_quantidade INT)
BEGIN
    UPDATE produtos
    SET estoque = estoque + p_quantidade
    WHERE id = p_produto_id;
    
    SELECT CONCAT('Estoque updated com sucesso. Foram adicionados ', p_quantidade, ' itens.') AS Mensagem;
END //
DELIMITER ;

CALL sp_adicionar_estoque(3, 5);

CREATE TABLE mv_resumo_vendas (
    produto_nome VARCHAR(100),
    total_vendido DECIMAL(10,2),
    ultima_atualizacao DATETIME
);

DELIMITER //
CREATE PROCEDURE sp_refresh_mv_resumo_vendas ()
BEGIN
    TRUNCATE TABLE mv_resumo_vendas;
    
    INSERT INTO mv_resumo_vendas (produto_nome, total_vendido, ultima_atualizacao)
    SELECT p.nome, SUM(v.total), CURRENT_TIMESTAMP
    FROM produtos p
    JOIN vendas v ON p.id = v.produto_id
    GROUP BY p.nome;
END //
DELIMITER ;

CALL sp_refresh_mv_resumo_vendas();

CREATE USER 'analista_dados'@'localhost' IDENTIFIED BY 'senha_segura_123';

GRANT SELECT ON lab_ecommerce.vw_produtos_disponiveis TO 'analista_dados'@'localhost';

FLUSH PRIVILEGES;

REVOKE SELECT ON lab_ecommerce.vw_produtos_disponiveis FROM 'analista_dados'@'localhost';
           
