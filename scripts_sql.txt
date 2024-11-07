CREATE SCHEMA dw AUTHORIZATION puc;

-- Tabela Dimensão dim_unidades
CREATE TABLE dw.dim_unidades (
    sk_unidades SERIAL PRIMARY KEY,
    cnes BIGINT NOT NULL,
    nome VARCHAR(255),
    logradouro VARCHAR(255),
    bairro VARCHAR(255)
);


-- Tabela Dimensão Data
CREATE TABLE dw.dim_data (
    sk_data SERIAL PRIMARY KEY,
    data_completa DATE,
    dia INT,
    mes INT,
    ano INT
);

-- Tabela Dimensão dim_equipes
CREATE TABLE dw.dim_equipes (
    sk_equipes SERIAL PRIMARY KEY,
    equipe_ine BIGINT NOT NULL,
    equipe_nome VARCHAR(255),
    equipe_tipo VARCHAR(255),
    data_ativacao_id INT,
    data_desativacao_id INT
);

-- Tabela Dimensão dim_profissionais
CREATE TABLE dw.dim_profissionais (
    sk_profissionais SERIAL PRIMARY KEY,
    profissional_cns BIGINT NOT NULL,
    profissional_cbo VARCHAR(255) NOT NULL,
    profissional_nome VARCHAR(255),
    data_entrada_equipe_id INT,
    data_desligamento_equipe_id INT
);

-- Tabela Fato atendimentos
CREATE TABLE dw.fato_atendimentos (
    sk_unidades INT NOT NULL,
    sk_data INT NOT NULL,
    sk_equipes INT NOT NULL,
    sk_profissionais INT NOT NULL,
    qt_atendimentos INT NOT NULL,
    PRIMARY KEY (sk_unidades, sk_data, sk_equipes, sk_profissionais),
    FOREIGN KEY (sk_unidades) REFERENCES dw.dim_unidades (sk_unidades),
    FOREIGN KEY (sk_data) REFERENCES dw.dim_data (sk_data),
    FOREIGN KEY (sk_equipes) REFERENCES dw.dim_equipes (sk_equipes),
    FOREIGN KEY (sk_profissionais) REFERENCES dw.dim_profissionais (sk_profissionais)
);

-- tratamento na tabela equipes do staging_area

select * from staging_area.equipes 

DELETE FROM staging_area.equipes
WHERE tipo_equipe = 'SEM EQUIPE VINCULADA';


UPDATE staging_area.equipes
SET equipe_dt_desativacao = NULL
WHERE equipe_dt_desativacao = '-';

SET datestyle = 'DMY';

ALTER TABLE staging_area.equipes 
ALTER COLUMN equipe_dt_desativacao SET DATA TYPE DATE USING equipe_dt_desativacao::DATE,
ALTER COLUMN equipe_dt_ativacao SET DATA TYPE DATE USING equipe_dt_ativacao::DATE,
ALTER COLUMN equipe_ine SET DATA TYPE BIGINT USING equipe_ine::BIGINT;

ALTER TABLE staging_area.equipes
ADD COLUMN data_ativacao_id INT;

UPDATE staging_area.equipes e
SET data_ativacao_id = sk_data
FROM dw.dim_data dt
WHERE e.equipe_dt_ativacao = dt.data_completa;

ALTER TABLE staging_area.equipes
ADD COLUMN data_desativacao_id INT;

UPDATE staging_area.equipes e
SET data_desativacao_id = sk_data
FROM dw.dim_data dt
WHERE e.equipe_dt_desativacao = dt.data_completa;

-- Tratamento da tabela de profissionais no schema staging_area
UPDATE staging_area.profissionais
SET equipe_dt_desligamento = NULL
WHERE equipe_dt_desligamento = '-';

UPDATE staging_area.profissionais
SET equipe_dt_entrada = NULL
WHERE equipe_dt_entrada = '-';


ALTER TABLE staging_area.profissionais 
ALTER COLUMN equipe_dt_entrada SET DATA TYPE DATE USING equipe_dt_entrada::DATE,
ALTER COLUMN equipe_dt_desligamento SET DATA TYPE DATE USING equipe_dt_desligamento::DATE;

ALTER TABLE staging_area.profissionais
ADD COLUMN data_entrada_equipe_id INT;

UPDATE staging_area.profissionais prof
SET data_entrada_equipe_id = sk_data 
FROM dw.dim_data dt
WHERE dt.data_completa = prof.equipe_dt_entrada;

ALTER TABLE staging_area.profissionais
ADD COLUMN data_desligamento_equipe_id INT;

UPDATE staging_area.profissionais prof
SET data_desligamento_equipe_id = sk_data
FROM dw.dim_data dt
WHERE dt.data_completa = prof.equipe_dt_desligamento;


-- Tratamento na tabela fato_producao no staging_area
select * from staging_area.atendimentos

ALTER TABLE staging_area.atendimentos
ALTER COLUMN nu_ine SET DATA TYPE BIGINT USING nu_ine::BIGINT;


-- Inserir dados na tabela dim_unidades
INSERT INTO dw.dim_unidades (cnes, nome, logradouro, bairro)
SELECT
    us.cnes,
    us.nome_fantasia,
    us.logradouro,
    us.bairro
FROM
    staging_area.unidades_saude us;
    
-- Carrega a tabela dim_data (tempo)
INSERT INTO dw.dim_data (ano, mes, dia, data_completa)
SELECT EXTRACT(YEAR FROM d)::INT, 
       EXTRACT(MONTH FROM d)::INT, 
       EXTRACT(DAY FROM d)::INT, d::DATE
FROM generate_series('2020-01-01'::DATE, '2025-12-31'::DATE, '1 day'::INTERVAL) d;

    
-- Inserir dados na tabela dim_equipes
INSERT INTO dw.dim_equipes (equipe_ine, equipe_nome, equipe_tipo,data_ativacao_id, 
                           data_desativacao_id)
SELECT
    e.equipe_ine,
    e.equipe_nome,
    e.tipo_equipe,
    e.data_ativacao_id,
    e.data_desativacao_id
FROM
    staging_area.equipes e;


-- Inserir dados na tabela dim_profissinais
INSERT INTO dw.dim_profissionais (profissional_cns, profissional_cbo, profissional_nome,
                                 data_entrada_equipe_id, data_desligamento_equipe_id)
SELECT
    p.profissional_cns,
    p.profissional_cbo,
    p.profissional_nome,
    p.data_entrada_equipe_id,
    p.data_desligamento_equipe_id
FROM
    staging_area.profissionais p;

-- Tratamento na tabela fato_atendimentos da staging_area
ALTER TABLE staging_area.atendimentos 
ALTER COLUMN dia SET DATA TYPE DATE USING dia::DATE,
ALTER COLUMN nu_ine SET DATA TYPE BIGINT USING nu_ine::BIGINT,
ALTER COLUMN qt_atendimentos SET DATA TYPE INTEGER USING qt_atendimentos::INTEGER;

UPDATE staging_area.atendimentos
SET no_profissional = UPPER(no_profissional);


CREATE EXTENSION IF NOT EXISTS pg_trgm;

ALTER TABLE staging_area.atendimentos
ADD COLUMN profissional_cns BIGINT;

UPDATE staging_area.atendimentos a
SET profissional_cns = p.profissional_cns
FROM staging_area.profissionais p
WHERE a.nu_cnes = p.cnes
AND similarity(a.no_profissional, p.profissional_nome) > 0.80;

INSERT INTO 
    dw.fato_atendimentos (sk_unidades, sk_data, sk_equipes, sk_profissionais,
                                  qt_atendimentos)                          
SELECT 
    sk_unidades,
    sk_data,
    sk_equipes,
    sk_profissionais,
    ROUND(SUM(qt_atendimentos)) AS quantidade_atendimentos
FROM 
    staging_area.atendimentos tb1, 
    dw.dim_unidades tb2,
    dw.dim_data tb3,
    dw.dim_equipes tb4,
    dw.dim_profissionais tb5
WHERE 
    tb2.cnes = tb1.nu_cnes
AND 
    tb3.data_completa = tb1.dia
AND 
    tb4.equipe_ine = tb1.nu_ine
AND 
    tb5.profissional_cns = tb1.profissional_cns
GROUP BY 
    sk_unidades, sk_data, sk_equipes, sk_profissionais;







