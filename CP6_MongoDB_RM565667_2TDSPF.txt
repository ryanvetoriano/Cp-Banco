// =====================================================================
//  FIAP · Mastering Relational and Non-Relational Database · CP6
//  Migração do projeto relacional (SQL) para NoSQL · MongoDB
//  Tema: Catálogo de produtos com atributos variados (item 02)
//
//  Alunos: Ryan Vetoriano (RM 565667) · Felipe Furlanetto (RM 562766)
//  Turma:  2TDSPF · Prof. Ms. Marcel Thomé Filho
//  Ambiente: MongoDB 8.0 · mongosh / MongoDB for VS Code (Playground)
// =====================================================================
//
//  MAPEAMENTO TABELA (SQL) -> COLLECTION (MongoDB)
//  ---------------------------------------------------------------------
//  Tabela SQL         Collection         Conversão para documento
//  categorias         categorias         PK -> _id; categoria pai nula vira campo ausente
//  marcas             marcas             PK -> _id
//  produtos           produtos           DECIMAL -> NumberDecimal; tags e imagens -> arrays
//  atributos_produto  atributos_produto  valor VARCHAR -> número ou texto; unidade nula omitida
//  variantes          variantes          colunas nulas (cor, tamanho, aro) omitidas
//  clientes           clientes           cidade/uf -> subdocumento endereco;
//                                        telefone_celular/telefone_fixo -> array telefones
//  avaliacoes         avaliacoes         DATETIME -> ISODate
//  estoque            estoque            PK -> _id; UNIQUE (sku, deposito) -> índice único
//  pedidos            pedidos            DECIMAL -> NumberDecimal; DATETIME -> ISODate
//  itens_pedido       itens_pedido       FK pedido_id mantida como referência
//  pagamentos         pagamentos         data_pagamento nula omitida
//
//  Chaves estrangeiras viram campos de referência (<entidade>_id) com índice.
//  Restrições NOT NULL / CHECK viram validadores $jsonSchema.
//  Restrições UNIQUE viram índices únicos.
// =====================================================================


// =====================================================================
//  0. CRIAÇÃO DO BANCO DE DADOS
// =====================================================================
use('cp6_catalogo');

// Garante uma execução limpa caso o script seja rodado novamente
db.dropDatabase();


// =====================================================================
//  1. ESTRUTURA · UMA COLLECTION PARA CADA TABELA
// =====================================================================

// ---------- categorias ----------
db.createCollection("categorias", {
  validator: { $jsonSchema: {
    bsonType: "object",
    required: ["_id", "slug", "nome"],
    properties: {
      _id:              { bsonType: "int" },
      slug:             { bsonType: "string", pattern: "^[a-z0-9/-]+$" },
      nome:             { bsonType: "string", minLength: 3 },
      categoria_pai_id: { bsonType: "int" }
    }
  } }
});

// ---------- marcas ----------
db.createCollection("marcas", {
  validator: { $jsonSchema: {
    bsonType: "object",
    required: ["_id", "nome", "pais_origem"],
    properties: {
      _id:         { bsonType: "int" },
      nome:        { bsonType: "string" },
      pais_origem: { bsonType: "string" }
    }
  } }
});

// ---------- produtos ----------
db.createCollection("produtos", {
  validator: { $jsonSchema: {
    bsonType: "object",
    required: ["_id", "slug", "nome", "categoria_id", "marca_id", "preco", "status"],
    properties: {
      _id:          { bsonType: "int" },
      slug:         { bsonType: "string", pattern: "^[a-z0-9-]+$" },
      nome:         { bsonType: "string", minLength: 3 },
      descricao:    { bsonType: "string" },
      categoria_id: { bsonType: "int" },
      marca_id:     { bsonType: "int" },
      preco:        { bsonType: "decimal", minimum: 0 },
      status:       { enum: ["ativo", "inativo", "rascunho"] },
      tags:         { bsonType: "array", items: { bsonType: "string" } },
      imagens:      { bsonType: "array", items: { bsonType: "string" } },
      versao:       { bsonType: "int", minimum: 1 },
      criado_em:    { bsonType: "date" },
      atualizado_em:{ bsonType: "date" }
    }
  } },
  validationLevel: "strict",   // valida todo insert e todo update
  validationAction: "error"    // rejeita o documento inválido
});

// ---------- atributos_produto ----------
db.createCollection("atributos_produto", {
  validator: { $jsonSchema: {
    bsonType: "object",
    required: ["_id", "produto_id", "chave", "valor"],
    properties: {
      _id:        { bsonType: "int" },
      produto_id: { bsonType: "int" },
      chave:      { bsonType: "string" },
      valor:      { bsonType: ["string", "int", "double"] },
      unidade:    { bsonType: "string" }
    }
  } }
});

// ---------- variantes ----------
db.createCollection("variantes", {
  validator: { $jsonSchema: {
    bsonType: "object",
    required: ["_id", "produto_id", "sku", "disponivel"],
    properties: {
      _id:        { bsonType: "int" },
      produto_id: { bsonType: "int" },
      sku:        { bsonType: "string" },
      cor:        { bsonType: "string" },
      tamanho:    { bsonType: "string" },
      aro:        { bsonType: "int" },
      disponivel: { bsonType: "bool" }
    }
  } }
});

// ---------- clientes ----------
db.createCollection("clientes", {
  validator: { $jsonSchema: {
    bsonType: "object",
    required: ["_id", "nome", "email", "cpf", "endereco"],
    properties: {
      _id:      { bsonType: "int" },
      nome:     { bsonType: "string" },
      email:    { bsonType: "string", pattern: "^.+@.+\\..+$" },
      cpf:      { bsonType: "string" },
      endereco: { bsonType: "object", required: ["cidade", "uf"],
                  properties: { cidade: { bsonType: "string" },
                                uf:     { bsonType: "string", minLength: 2, maxLength: 2 } } },
      telefones:{ bsonType: "array",
                  items: { bsonType: "object", required: ["tipo", "numero"],
                           properties: { tipo: { enum: ["celular", "fixo"] } } } },
      data_cadastro: { bsonType: "date" }
    }
  } }
});

// ---------- avaliacoes ----------
db.createCollection("avaliacoes", {
  validator: { $jsonSchema: {
    bsonType: "object",
    required: ["_id", "produto_id", "cliente_id", "nota", "status"],
    properties: {
      _id:            { bsonType: "int" },
      produto_id:     { bsonType: "int" },
      cliente_id:     { bsonType: "int" },
      nota:           { bsonType: "int", minimum: 1, maximum: 5 },
      comentario:     { bsonType: "string" },
      status:         { enum: ["aprovada", "pendente", "reprovada"] },
      data_avaliacao: { bsonType: "date" }
    }
  } }
});

// ---------- estoque ----------
db.createCollection("estoque", {
  validator: { $jsonSchema: {
    bsonType: "object",
    required: ["_id", "sku", "deposito", "quantidade"],
    properties: {
      _id:           { bsonType: "int" },
      sku:           { bsonType: "string" },
      deposito:      { bsonType: "string" },
      quantidade:    { bsonType: "int", minimum: 0 },
      atualizado_em: { bsonType: "date" }
    }
  } }
});

// ---------- pedidos ----------
db.createCollection("pedidos", {
  validator: { $jsonSchema: {
    bsonType: "object",
    required: ["_id", "cliente_id", "data_pedido", "status", "valor_total"],
    properties: {
      _id:         { bsonType: "int" },
      cliente_id:  { bsonType: "int" },
      data_pedido: { bsonType: "date" },
      status:      { enum: ["aguardando_pagamento", "pago", "enviado", "entregue", "cancelado"] },
      canal:       { enum: ["site", "app"] },
      valor_total: { bsonType: "decimal", minimum: 0 }
    }
  } }
});

// ---------- itens_pedido ----------
db.createCollection("itens_pedido", {
  validator: { $jsonSchema: {
    bsonType: "object",
    required: ["_id", "pedido_id", "sku", "quantidade", "preco_unitario"],
    properties: {
      _id:            { bsonType: "int" },
      pedido_id:      { bsonType: "int" },
      sku:            { bsonType: "string" },
      quantidade:     { bsonType: "int", minimum: 1 },
      preco_unitario: { bsonType: "decimal", minimum: 0 }
    }
  } }
});

// ---------- pagamentos ----------
db.createCollection("pagamentos", {
  validator: { $jsonSchema: {
    bsonType: "object",
    required: ["_id", "pedido_id", "metodo", "valor", "status"],
    properties: {
      _id:            { bsonType: "int" },
      pedido_id:      { bsonType: "int" },
      metodo:         { enum: ["pix", "cartao_credito", "boleto"] },
      valor:          { bsonType: "decimal", minimum: 0 },
      parcelas:       { bsonType: "int", minimum: 1, maximum: 12 },
      status:         { enum: ["aprovado", "pendente", "estornado"] },
      data_pagamento: { bsonType: "date" }
    }
  } }
});

// ---------- índices (equivalentes às PK/UNIQUE/FK do modelo SQL) ----------
db.categorias.createIndex({ slug: 1 }, { unique: true });
db.marcas.createIndex({ nome: 1 }, { unique: true });
db.produtos.createIndex({ slug: 1 }, { unique: true });
db.produtos.createIndex({ categoria_id: 1, status: 1, preco: 1 });   // regra ESR
db.produtos.createIndex({ marca_id: 1 });
db.atributos_produto.createIndex({ produto_id: 1, chave: 1 }, { unique: true });
db.variantes.createIndex({ sku: 1 }, { unique: true });
db.variantes.createIndex({ produto_id: 1 });
db.clientes.createIndex({ email: 1 }, { unique: true });
db.clientes.createIndex({ cpf: 1 }, { unique: true });
db.avaliacoes.createIndex({ produto_id: 1, data_avaliacao: -1 });
db.estoque.createIndex({ sku: 1, deposito: 1 }, { unique: true });
db.pedidos.createIndex({ cliente_id: 1, data_pedido: -1 });
db.itens_pedido.createIndex({ pedido_id: 1 });
db.pagamentos.createIndex({ pedido_id: 1 }, { unique: true });

// Conferência da estrutura criada
db.getCollectionNames();


// =====================================================================
//  2. INSERÇÃO DOS DADOS · CADA LINHA DA TABELA VIRA UM DOCUMENTO
// =====================================================================

// ---------- categorias ----------
// Categorias raiz não têm o campo categoria_pai_id (no SQL era NULL)
db.categorias.insertMany([
  { _id: 1, slug: "vestuario",               nome: "Vestuário" },
  { _id: 2, slug: "vestuario/camisetas",     nome: "Camisetas",   categoria_pai_id: 1 },
  { _id: 3, slug: "eletronicos",             nome: "Eletrônicos" },
  { _id: 4, slug: "eletronicos/notebooks",   nome: "Notebooks",   categoria_pai_id: 3 },
  { _id: 5, slug: "eletronicos/smartphones", nome: "Smartphones", categoria_pai_id: 3 },
  { _id: 6, slug: "joias",                   nome: "Joias" },
  { _id: 7, slug: "joias/aneis",             nome: "Anéis",       categoria_pai_id: 6 },
  { _id: 8, slug: "calcados",                nome: "Calçados" },
  { _id: 9, slug: "calcados/tenis",          nome: "Tênis",       categoria_pai_id: 8 }
]);

// ---------- marcas ----------
db.marcas.insertMany([
  { _id: 1, nome: "Aurora",       pais_origem: "Brasil" },
  { _id: 2, nome: "Lumen Tech",   pais_origem: "Brasil" },
  { _id: 3, nome: "Nimbus",       pais_origem: "Estados Unidos" },
  { _id: 4, nome: "Prata Fina",   pais_origem: "Brasil" },
  { _id: 5, nome: "Passo Firme",  pais_origem: "Brasil" },
  { _id: 6, nome: "Orion Mobile", pais_origem: "Coreia do Sul" }
]);

// ---------- produtos ----------
db.produtos.insertMany([
  { _id: 1, slug: "camiseta-basica-algodao", nome: "Camiseta Básica de Algodão",
    descricao: "Malha 100% algodão, corte reto e gola careca.",
    categoria_id: 2, marca_id: 1, preco: NumberDecimal("79.90"), status: "ativo",
    tags: ["algodao", "basica", "unissex"],
    imagens: ["https://cdn.exemplo.com/cam-001-1.webp"],
    versao: 7, criado_em: ISODate("2026-03-10T10:00:00Z"), atualizado_em: ISODate("2026-10-18T12:00:00Z") },
  { _id: 2, slug: "camiseta-dry-fit", nome: "Camiseta Dry Fit Esportiva",
    descricao: "Tecido leve de secagem rápida com proteção UV.",
    categoria_id: 2, marca_id: 1, preco: NumberDecimal("99.90"), status: "ativo",
    tags: ["esporte", "poliester", "unissex"],
    imagens: ["https://cdn.exemplo.com/cam-002-1.webp"],
    versao: 3, criado_em: ISODate("2026-04-02T10:00:00Z"), atualizado_em: ISODate("2026-09-30T15:00:00Z") },
  { _id: 3, slug: "camiseta-estampada-retro", nome: "Camiseta Estampada Retrô",
    descricao: "Estampa vintage em silk, malha penteada.",
    categoria_id: 2, marca_id: 1, preco: NumberDecimal("89.90"), status: "rascunho",
    tags: ["estampada", "algodao"],
    imagens: ["https://cdn.exemplo.com/cam-003-1.webp"],
    versao: 1, criado_em: ISODate("2026-10-12T09:00:00Z"), atualizado_em: ISODate("2026-10-12T09:00:00Z") },
  { _id: 4, slug: "notebook-pro-14", nome: "Notebook Pro 14",
    descricao: "Notebook 14 polegadas com 16 GB de RAM e SSD de 512 GB.",
    categoria_id: 4, marca_id: 3, preco: NumberDecimal("6899.00"), status: "ativo",
    tags: ["16gb", "ssd", "bivolt"],
    imagens: ["https://cdn.exemplo.com/nb-pro14-1.webp", "https://cdn.exemplo.com/nb-pro14-2.webp"],
    versao: 4, criado_em: ISODate("2026-02-20T10:00:00Z"), atualizado_em: ISODate("2026-10-01T10:00:00Z") },
  { _id: 5, slug: "notebook-air-13", nome: "Notebook Air 13",
    descricao: "Ultrafino de 13,3 polegadas, ideal para estudo e trabalho.",
    categoria_id: 4, marca_id: 2, preco: NumberDecimal("4599.00"), status: "ativo",
    tags: ["8gb", "ssd", "leve"],
    imagens: ["https://cdn.exemplo.com/nb-air13-1.webp"],
    versao: 2, criado_em: ISODate("2026-05-15T10:00:00Z"), atualizado_em: ISODate("2026-08-22T10:00:00Z") },
  { _id: 6, slug: "smartphone-orion-x", nome: "Smartphone Orion X 256GB",
    descricao: "Tela de 6,7 polegadas, 5G e câmera tripla.",
    categoria_id: 5, marca_id: 6, preco: NumberDecimal("3299.00"), status: "ativo",
    tags: ["5g", "256gb", "dual-chip"],
    imagens: ["https://cdn.exemplo.com/sm-orx-1.webp"],
    versao: 5, criado_em: ISODate("2026-01-08T10:00:00Z"), atualizado_em: ISODate("2026-09-18T10:00:00Z") },
  { _id: 7, slug: "smartphone-orion-lite", nome: "Smartphone Orion Lite 128GB",
    descricao: "Modelo de entrada com tela de 6,1 polegadas.",
    categoria_id: 5, marca_id: 6, preco: NumberDecimal("1499.00"), status: "inativo",
    tags: ["4g", "128gb"],
    imagens: ["https://cdn.exemplo.com/sm-orl-1.webp"],
    versao: 9, criado_em: ISODate("2025-06-01T10:00:00Z"), atualizado_em: ISODate("2026-07-01T10:00:00Z") },
  { _id: 8, slug: "anel-solitario-prata-925", nome: "Anel Solitário Prata 925",
    descricao: "Anel em prata 925 com zircônia de 6 mm.",
    categoria_id: 7, marca_id: 4, preco: NumberDecimal("249.90"), status: "ativo",
    tags: ["prata", "zirconia", "presente"],
    imagens: ["https://cdn.exemplo.com/anl-925-1.webp"],
    versao: 2, criado_em: ISODate("2026-03-25T10:00:00Z"), atualizado_em: ISODate("2026-09-01T10:00:00Z") },
  { _id: 9, slug: "anel-aparador-ouro-18k", nome: "Anel Aparador Ouro 18k",
    descricao: "Aparador liso em ouro amarelo 18k.",
    categoria_id: 7, marca_id: 4, preco: NumberDecimal("1890.00"), status: "ativo",
    tags: ["ouro", "presente"],
    imagens: ["https://cdn.exemplo.com/anl-ou18-1.webp"],
    versao: 1, criado_em: ISODate("2026-06-10T10:00:00Z"), atualizado_em: ISODate("2026-06-10T10:00:00Z") },
  { _id: 10, slug: "tenis-corrida-leve", nome: "Tênis de Corrida Leve",
    descricao: "Cabedal em mesh respirável e entressola com amortecimento.",
    categoria_id: 9, marca_id: 5, preco: NumberDecimal("399.90"), status: "ativo",
    tags: ["corrida", "amortecimento", "unissex"],
    imagens: ["https://cdn.exemplo.com/ten-cor-1.webp"],
    versao: 3, criado_em: ISODate("2026-04-18T10:00:00Z"), atualizado_em: ISODate("2026-10-05T10:00:00Z") }
]);

// ---------- atributos_produto ----------
// Padrão Attribute: pares chave/valor; valores numéricos deixam de ser texto
db.atributos_produto.insertMany([
  { _id: 1,  produto_id: 1,  chave: "material",       valor: "algodão" },
  { _id: 2,  produto_id: 1,  chave: "gramatura",      valor: 160,   unidade: "g/m²" },
  { _id: 3,  produto_id: 1,  chave: "gola",           valor: "careca" },
  { _id: 4,  produto_id: 2,  chave: "material",       valor: "poliéster" },
  { _id: 5,  produto_id: 2,  chave: "gramatura",      valor: 120,   unidade: "g/m²" },
  { _id: 6,  produto_id: 2,  chave: "protecao_uv",    valor: 50,    unidade: "FPS" },
  { _id: 7,  produto_id: 3,  chave: "material",       valor: "algodão" },
  { _id: 8,  produto_id: 3,  chave: "estampa",        valor: "retrô" },
  { _id: 9,  produto_id: 4,  chave: "memoria_ram",    valor: 16,    unidade: "GB" },
  { _id: 10, produto_id: 4,  chave: "armazenamento",  valor: 512,   unidade: "GB" },
  { _id: 11, produto_id: 4,  chave: "tela",           valor: 14,    unidade: "pol" },
  { _id: 12, produto_id: 4,  chave: "voltagem",       valor: "bivolt" },
  { _id: 13, produto_id: 5,  chave: "memoria_ram",    valor: 8,     unidade: "GB" },
  { _id: 14, produto_id: 5,  chave: "armazenamento",  valor: 256,   unidade: "GB" },
  { _id: 15, produto_id: 5,  chave: "tela",           valor: 13.3,  unidade: "pol" },
  { _id: 16, produto_id: 5,  chave: "voltagem",       valor: "bivolt" },
  { _id: 17, produto_id: 6,  chave: "armazenamento",  valor: 256,   unidade: "GB" },
  { _id: 18, produto_id: 6,  chave: "tela",           valor: 6.7,   unidade: "pol" },
  { _id: 19, produto_id: 6,  chave: "rede",           valor: "5G" },
  { _id: 20, produto_id: 7,  chave: "armazenamento",  valor: 128,   unidade: "GB" },
  { _id: 21, produto_id: 7,  chave: "tela",           valor: 6.1,   unidade: "pol" },
  { _id: 22, produto_id: 7,  chave: "rede",           valor: "4G" },
  { _id: 23, produto_id: 8,  chave: "material",       valor: "prata 925" },
  { _id: 24, produto_id: 8,  chave: "pedra",          valor: "zircônia" },
  { _id: 25, produto_id: 8,  chave: "pedra_diametro", valor: 6,     unidade: "mm" },
  { _id: 26, produto_id: 9,  chave: "material",       valor: "ouro 18k" },
  { _id: 27, produto_id: 10, chave: "drop",           valor: 8,     unidade: "mm" },
  { _id: 28, produto_id: 10, chave: "peso",           valor: 230,   unidade: "g" },
  { _id: 29, produto_id: 10, chave: "solado",         valor: "borracha" }
]);

// ---------- variantes ----------
// No SQL, cor/tamanho/aro ficavam NULL conforme a categoria; aqui o campo simplesmente não existe
db.variantes.insertMany([
  { _id: 1,  produto_id: 1,  sku: "CAM-001-P-PRT",  cor: "preto",  tamanho: "P",  disponivel: true },
  { _id: 2,  produto_id: 1,  sku: "CAM-001-M-PRT",  cor: "preto",  tamanho: "M",  disponivel: true },
  { _id: 3,  produto_id: 1,  sku: "CAM-001-M-BRC",  cor: "branco", tamanho: "M",  disponivel: false },
  { _id: 4,  produto_id: 2,  sku: "CAM-002-M-AZL",  cor: "azul",   tamanho: "M",  disponivel: true },
  { _id: 5,  produto_id: 2,  sku: "CAM-002-G-AZL",  cor: "azul",   tamanho: "G",  disponivel: true },
  { _id: 6,  produto_id: 3,  sku: "CAM-003-M-BGE",  cor: "bege",   tamanho: "M",  disponivel: false },
  { _id: 7,  produto_id: 4,  sku: "NB-PRO14-CNZ",   cor: "cinza",                 disponivel: true },
  { _id: 8,  produto_id: 4,  sku: "NB-PRO14-PRT",   cor: "preto",                 disponivel: true },
  { _id: 9,  produto_id: 5,  sku: "NB-AIR13-PRA",   cor: "prata",                 disponivel: true },
  { _id: 10, produto_id: 6,  sku: "SM-ORX-PRT",     cor: "preto",                 disponivel: true },
  { _id: 11, produto_id: 6,  sku: "SM-ORX-VRD",     cor: "verde",                 disponivel: false },
  { _id: 12, produto_id: 7,  sku: "SM-ORL-AZL",     cor: "azul",                  disponivel: true },
  { _id: 13, produto_id: 8,  sku: "ANL-925-14",     aro: 14,                      disponivel: true },
  { _id: 14, produto_id: 8,  sku: "ANL-925-16",     aro: 16,                      disponivel: true },
  { _id: 15, produto_id: 8,  sku: "ANL-925-18",     aro: 18,                      disponivel: false },
  { _id: 16, produto_id: 9,  sku: "ANL-OU18-16",    aro: 16,                      disponivel: true },
  { _id: 17, produto_id: 10, sku: "TEN-COR-39-PRT", cor: "preto",  tamanho: "39", disponivel: true },
  { _id: 18, produto_id: 10, sku: "TEN-COR-41-PRT", cor: "preto",  tamanho: "41", disponivel: true },
  { _id: 19, produto_id: 10, sku: "TEN-COR-41-BRC", cor: "branco", tamanho: "41", disponivel: false }
]);

// ---------- clientes ----------
// cidade/uf viram o subdocumento endereco; os dois telefones viram um array
db.clientes.insertMany([
  { _id: 1, nome: "Ana Souza", email: "ana.souza@email.com", cpf: "100.200.300-01",
    endereco: { cidade: "São Paulo", uf: "SP" },
    telefones: [ { tipo: "celular", numero: "11 98765-1001" }, { tipo: "fixo", numero: "11 3456-1001" } ],
    data_cadastro: ISODate("2025-03-14T10:00:00Z") },
  { _id: 2, nome: "Bruno Lima", email: "bruno.lima@empresa.com.br", cpf: "100.200.300-02",
    endereco: { cidade: "Rio de Janeiro", uf: "RJ" },
    telefones: [ { tipo: "celular", numero: "21 99876-1002" } ],
    data_cadastro: ISODate("2025-07-02T10:00:00Z") },
  { _id: 3, nome: "Carla Mendes", email: "carla.mendes@email.com", cpf: "100.200.300-03",
    endereco: { cidade: "Belo Horizonte", uf: "MG" },
    telefones: [ { tipo: "celular", numero: "31 98888-1003" }, { tipo: "fixo", numero: "31 3222-1003" } ],
    data_cadastro: ISODate("2025-09-20T10:00:00Z") },
  { _id: 4, nome: "Diego Rocha", email: "diego.rocha@email.com", cpf: "100.200.300-04",
    endereco: { cidade: "Campinas", uf: "SP" },
    telefones: [ { tipo: "celular", numero: "19 99777-1004" } ],
    data_cadastro: ISODate("2026-01-11T10:00:00Z") },
  { _id: 5, nome: "Elisa Martins", email: "elisa.martins@consultoria.com.br", cpf: "100.200.300-05",
    endereco: { cidade: "Curitiba", uf: "PR" },
    telefones: [ { tipo: "celular", numero: "41 99666-1005" } ],
    data_cadastro: ISODate("2026-02-08T10:00:00Z") },
  { _id: 6, nome: "Fábio Andrade", email: "fabio.andrade@email.com", cpf: "100.200.300-06",
    endereco: { cidade: "Porto Alegre", uf: "RS" },
    telefones: [ { tipo: "celular", numero: "51 99555-1006" }, { tipo: "fixo", numero: "51 3333-1006" } ],
    data_cadastro: ISODate("2026-03-30T10:00:00Z") },
  { _id: 7, nome: "Gabriela Nunes", email: "gabriela.nunes@email.com", cpf: "100.200.300-07",
    endereco: { cidade: "Salvador", uf: "BA" },
    telefones: [ { tipo: "celular", numero: "71 99444-1007" } ],
    data_cadastro: ISODate("2026-05-17T10:00:00Z") },
  { _id: 8, nome: "Henrique Costa", email: "henrique.costa@empresa.com.br", cpf: "100.200.300-08",
    endereco: { cidade: "São Paulo", uf: "SP" },
    telefones: [ { tipo: "celular", numero: "11 99333-1008" } ],
    data_cadastro: ISODate("2026-08-03T10:00:00Z") }
]);

// ---------- avaliacoes ----------
db.avaliacoes.insertMany([
  { _id: 1,  produto_id: 1,  cliente_id: 1, nota: 5, comentario: "Tecido macio e caimento ótimo.",
    status: "aprovada",  data_avaliacao: ISODate("2026-09-10T18:20:00Z") },
  { _id: 2,  produto_id: 1,  cliente_id: 3, nota: 4, comentario: "Boa camiseta, encolheu um pouco.",
    status: "aprovada",  data_avaliacao: ISODate("2026-09-18T21:05:00Z") },
  { _id: 3,  produto_id: 4,  cliente_id: 2, nota: 5, comentario: "Rápido e silencioso, a bateria dura o dia todo.",
    status: "aprovada",  data_avaliacao: ISODate("2026-09-15T13:40:00Z") },
  { _id: 4,  produto_id: 4,  cliente_id: 4, nota: 2, comentario: "Esquenta bastante em jogos.",
    status: "aprovada",  data_avaliacao: ISODate("2026-09-28T22:10:00Z") },
  { _id: 5,  produto_id: 8,  cliente_id: 1, nota: 5, comentario: "Lindo, chegou numa caixinha de presente.",
    status: "aprovada",  data_avaliacao: ISODate("2026-09-12T11:00:00Z") },
  { _id: 6,  produto_id: 10, cliente_id: 3, nota: 4, comentario: "Confortável para corridas longas.",
    status: "aprovada",  data_avaliacao: ISODate("2026-09-25T07:30:00Z") },
  { _id: 7,  produto_id: 6,  cliente_id: 4, nota: 1, comentario: "Compre já!!! Link na bio.",
    status: "reprovada", data_avaliacao: ISODate("2026-09-22T03:12:00Z") },
  { _id: 8,  produto_id: 2,  cliente_id: 8, nota: 3, comentario: "Seca rápido, mas o tamanho é pequeno.",
    status: "pendente",  data_avaliacao: ISODate("2026-10-18T19:45:00Z") },
  { _id: 9,  produto_id: 9,  cliente_id: 6, nota: 5, comentario: "Acabamento impecável.",
    status: "aprovada",  data_avaliacao: ISODate("2026-10-14T16:00:00Z") },
  { _id: 10, produto_id: 6,  cliente_id: 1, nota: 4, comentario: "Câmera excelente, bateria mediana.",
    status: "aprovada",  data_avaliacao: ISODate("2026-10-06T20:30:00Z") },
  { _id: 11, produto_id: 2,  cliente_id: 8, nota: 1, comentario: "Promoção imperdível em outro site.",
    status: "reprovada", data_avaliacao: ISODate("2026-10-19T02:58:00Z") },
  { _id: 12, produto_id: 10, cliente_id: 8, nota: 2, comentario: "Achei duro, não gostei.",
    status: "pendente",  data_avaliacao: ISODate("2026-10-20T12:15:00Z") }
]);

// ---------- estoque ----------
db.estoque.insertMany([
  { _id: 1,  sku: "CAM-001-P-PRT",  deposito: "CD-Barueri", quantidade: 35, atualizado_em: ISODate("2026-10-20T06:00:00Z") },
  { _id: 2,  sku: "CAM-001-M-PRT",  deposito: "CD-Barueri", quantidade: 42, atualizado_em: ISODate("2026-10-20T06:00:00Z") },
  { _id: 3,  sku: "CAM-001-M-BRC",  deposito: "CD-Barueri", quantidade: 0,  atualizado_em: ISODate("2026-10-20T06:00:00Z") },
  { _id: 4,  sku: "CAM-002-M-AZL",  deposito: "CD-Barueri", quantidade: 28, atualizado_em: ISODate("2026-10-20T06:00:00Z") },
  { _id: 5,  sku: "CAM-002-G-AZL",  deposito: "CD-Recife",  quantidade: 20, atualizado_em: ISODate("2026-10-20T06:00:00Z") },
  { _id: 6,  sku: "CAM-003-M-BGE",  deposito: "CD-Barueri", quantidade: 0,  atualizado_em: ISODate("2026-10-20T06:00:00Z") },
  { _id: 7,  sku: "NB-PRO14-CNZ",   deposito: "CD-Barueri", quantidade: 12, atualizado_em: ISODate("2026-10-20T06:00:00Z") },
  { _id: 8,  sku: "NB-PRO14-PRT",   deposito: "CD-Barueri", quantidade: 7,  atualizado_em: ISODate("2026-10-20T06:00:00Z") },
  { _id: 9,  sku: "NB-AIR13-PRA",   deposito: "CD-Barueri", quantidade: 3,  atualizado_em: ISODate("2026-10-20T06:00:00Z") },
  { _id: 10, sku: "SM-ORX-PRT",     deposito: "CD-Barueri", quantidade: 15, atualizado_em: ISODate("2026-10-20T06:00:00Z") },
  { _id: 11, sku: "SM-ORX-VRD",     deposito: "CD-Recife",  quantidade: 0,  atualizado_em: ISODate("2026-10-20T06:00:00Z") },
  { _id: 12, sku: "SM-ORL-AZL",     deposito: "CD-Recife",  quantidade: 9,  atualizado_em: ISODate("2026-10-20T06:00:00Z") },
  { _id: 13, sku: "ANL-925-14",     deposito: "CD-Barueri", quantidade: 6,  atualizado_em: ISODate("2026-10-20T06:00:00Z") },
  { _id: 14, sku: "ANL-925-16",     deposito: "CD-Barueri", quantidade: 11, atualizado_em: ISODate("2026-10-20T06:00:00Z") },
  { _id: 15, sku: "ANL-925-18",     deposito: "CD-Barueri", quantidade: 0,  atualizado_em: ISODate("2026-10-20T06:00:00Z") },
  { _id: 16, sku: "ANL-OU18-16",    deposito: "CD-Barueri", quantidade: 2,  atualizado_em: ISODate("2026-10-20T06:00:00Z") },
  { _id: 17, sku: "TEN-COR-39-PRT", deposito: "CD-Recife",  quantidade: 22, atualizado_em: ISODate("2026-10-20T06:00:00Z") },
  { _id: 18, sku: "TEN-COR-41-PRT", deposito: "CD-Recife",  quantidade: 14, atualizado_em: ISODate("2026-10-20T06:00:00Z") },
  { _id: 19, sku: "TEN-COR-41-BRC", deposito: "CD-Recife",  quantidade: 0,  atualizado_em: ISODate("2026-10-20T06:00:00Z") }
]);

// ---------- pedidos ----------
db.pedidos.insertMany([
  { _id: 1, cliente_id: 1, data_pedido: ISODate("2026-09-02T14:30:00Z"), status: "entregue",
    canal: "site", valor_total: NumberDecimal("409.70") },
  { _id: 2, cliente_id: 2, data_pedido: ISODate("2026-09-05T20:05:00Z"), status: "entregue",
    canal: "app",  valor_total: NumberDecimal("6899.00") },
  { _id: 3, cliente_id: 3, data_pedido: ISODate("2026-09-12T09:40:00Z"), status: "enviado",
    canal: "site", valor_total: NumberDecimal("499.80") },
  { _id: 4, cliente_id: 4, data_pedido: ISODate("2026-09-20T16:00:00Z"), status: "cancelado",
    canal: "app",  valor_total: NumberDecimal("3299.00") },
  { _id: 5, cliente_id: 1, data_pedido: ISODate("2026-10-01T11:15:00Z"), status: "pago",
    canal: "app",  valor_total: NumberDecimal("3378.90") },
  { _id: 6, cliente_id: 5, data_pedido: ISODate("2026-10-08T17:50:00Z"), status: "aguardando_pagamento",
    canal: "site", valor_total: NumberDecimal("4599.00") },
  { _id: 7, cliente_id: 6, data_pedido: ISODate("2026-10-10T19:30:00Z"), status: "entregue",
    canal: "site", valor_total: NumberDecimal("1890.00") },
  { _id: 8, cliente_id: 8, data_pedido: ISODate("2026-10-15T08:10:00Z"), status: "enviado",
    canal: "app",  valor_total: NumberDecimal("699.60") }
]);

// ---------- itens_pedido ----------
db.itens_pedido.insertMany([
  { _id: 1,  pedido_id: 1, sku: "CAM-001-M-PRT",  quantidade: 2, preco_unitario: NumberDecimal("79.90") },
  { _id: 2,  pedido_id: 1, sku: "ANL-925-16",     quantidade: 1, preco_unitario: NumberDecimal("249.90") },
  { _id: 3,  pedido_id: 2, sku: "NB-PRO14-CNZ",   quantidade: 1, preco_unitario: NumberDecimal("6899.00") },
  { _id: 4,  pedido_id: 3, sku: "TEN-COR-39-PRT", quantidade: 1, preco_unitario: NumberDecimal("399.90") },
  { _id: 5,  pedido_id: 3, sku: "CAM-002-M-AZL",  quantidade: 1, preco_unitario: NumberDecimal("99.90") },
  { _id: 6,  pedido_id: 4, sku: "SM-ORX-PRT",     quantidade: 1, preco_unitario: NumberDecimal("3299.00") },
  { _id: 7,  pedido_id: 5, sku: "SM-ORX-PRT",     quantidade: 1, preco_unitario: NumberDecimal("3299.00") },
  { _id: 8,  pedido_id: 5, sku: "CAM-001-P-PRT",  quantidade: 1, preco_unitario: NumberDecimal("79.90") },
  { _id: 9,  pedido_id: 6, sku: "NB-AIR13-PRA",   quantidade: 1, preco_unitario: NumberDecimal("4599.00") },
  { _id: 10, pedido_id: 7, sku: "ANL-OU18-16",    quantidade: 1, preco_unitario: NumberDecimal("1890.00") },
  { _id: 11, pedido_id: 8, sku: "CAM-002-G-AZL",  quantidade: 3, preco_unitario: NumberDecimal("99.90") },
  { _id: 12, pedido_id: 8, sku: "TEN-COR-41-PRT", quantidade: 1, preco_unitario: NumberDecimal("399.90") }
]);

// ---------- pagamentos ----------
// Pagamento pendente não tem data_pagamento (no SQL era NULL)
db.pagamentos.insertMany([
  { _id: 1, pedido_id: 1, metodo: "pix",            valor: NumberDecimal("409.70"),  parcelas: 1,  status: "aprovado",
    data_pagamento: ISODate("2026-09-02T14:35:00Z") },
  { _id: 2, pedido_id: 2, metodo: "cartao_credito", valor: NumberDecimal("6899.00"), parcelas: 10, status: "aprovado",
    data_pagamento: ISODate("2026-09-05T20:12:00Z") },
  { _id: 3, pedido_id: 3, metodo: "cartao_credito", valor: NumberDecimal("499.80"),  parcelas: 2,  status: "aprovado",
    data_pagamento: ISODate("2026-09-12T09:47:00Z") },
  { _id: 4, pedido_id: 4, metodo: "cartao_credito", valor: NumberDecimal("3299.00"), parcelas: 6,  status: "estornado",
    data_pagamento: ISODate("2026-09-20T16:05:00Z") },
  { _id: 5, pedido_id: 5, metodo: "pix",            valor: NumberDecimal("3378.90"), parcelas: 1,  status: "aprovado",
    data_pagamento: ISODate("2026-10-01T11:22:00Z") },
  { _id: 6, pedido_id: 6, metodo: "boleto",         valor: NumberDecimal("4599.00"), parcelas: 1,  status: "pendente" },
  { _id: 7, pedido_id: 7, metodo: "cartao_credito", valor: NumberDecimal("1890.00"), parcelas: 3,  status: "aprovado",
    data_pagamento: ISODate("2026-10-10T19:40:00Z") },
  { _id: 8, pedido_id: 8, metodo: "pix",            valor: NumberDecimal("699.60"),  parcelas: 1,  status: "aprovado",
    data_pagamento: ISODate("2026-10-15T08:15:00Z") }
]);

// Conferência: total de documentos por collection
db.getCollectionNames().sort().map(c => ({ collection: c, documentos: db.getCollection(c).countDocuments() }));


// =====================================================================
//  3. UPDATES · COLLECTION ESCOLHIDA: produtos
// =====================================================================

// ---------- 3.1 updateOne ----------
// Back-office: promoção da camiseta básica com controle de concorrência otimista.
// Só altera se ninguém salvou o produto desde a leitura (versao 7).
// SQL: UPDATE produtos SET preco = 74.90, versao = versao + 1, atualizado_em = NOW()
//      WHERE slug = 'camiseta-basica-algodao' AND versao = 7;
db.produtos.find({ slug: "camiseta-basica-algodao" }, { nome: 1, preco: 1, tags: 1, versao: 1 });   // antes

db.produtos.updateOne(
  { slug: "camiseta-basica-algodao", versao: 7 },
  {
    $set:      { preco: NumberDecimal("74.90"), atualizado_em: new Date() },
    $addToSet: { tags: "promocao" },
    $inc:      { versao: 1 }
  }
);

db.produtos.find({ slug: "camiseta-basica-algodao" }, { nome: 1, preco: 1, tags: 1, versao: 1 });   // depois

// ---------- 3.2 updateMany ----------
// Reajuste de 5% em todos os notebooks ativos (categoria_id 4).
// SQL: UPDATE produtos SET preco = preco * 1.05, versao = versao + 1, atualizado_em = NOW()
//      WHERE categoria_id = 4 AND status = 'ativo';
db.produtos.find({ categoria_id: 4, status: "ativo" }, { nome: 1, preco: 1, versao: 1 });   // antes

db.produtos.updateMany(
  { categoria_id: 4, status: "ativo" },
  {
    $mul: { preco: NumberDecimal("1.05") },
    $inc: { versao: 1 },
    $set: { atualizado_em: new Date() }
  }
);

db.produtos.find({ categoria_id: 4, status: "ativo" }, { nome: 1, preco: 1, versao: 1 });   // depois


// =====================================================================
//  4. DELETES · COLLECTION ESCOLHIDA: avaliacoes
// =====================================================================

// ---------- 4.1 deleteOne ----------
// O cliente 8 pediu a remoção da avaliação 12.
// SQL: DELETE FROM avaliacoes WHERE id_avaliacao = 12;
db.avaliacoes.countDocuments();   // antes: 12

db.avaliacoes.deleteOne({ _id: 12 });

db.avaliacoes.countDocuments();   // depois: 11

// ---------- 4.2 deleteMany ----------
// Remove todas as avaliações reprovadas pela moderação (spam).
// SQL: DELETE FROM avaliacoes WHERE status = 'reprovada';
db.avaliacoes.find({ status: "reprovada" }, { comentario: 1, status: 1 });   // antes: 2 documentos

db.avaliacoes.deleteMany({ status: "reprovada" });

db.avaliacoes.countDocuments();   // depois: 9


// =====================================================================
//  5. RELATÓRIOS (find) · DOIS POR OPERADOR
// =====================================================================

// ---------------------------------------------------------------------
//  $eq · igual a
// ---------------------------------------------------------------------
// R01 · Produtos da categoria Notebooks
// SQL: SELECT nome, preco FROM produtos WHERE categoria_id = 4;
db.produtos.find({ categoria_id: { $eq: 4 } }, { _id: 0, nome: 1, preco: 1 });

// R02 · Pedidos já entregues
// SQL: SELECT * FROM pedidos WHERE status = 'entregue';
db.pedidos.find({ status: { $eq: "entregue" } }, { _id: 1, cliente_id: 1, data_pedido: 1, valor_total: 1 });

// ---------------------------------------------------------------------
//  $ne · diferente de
// ---------------------------------------------------------------------
// R03 · Produtos fora da vitrine (status diferente de ativo)
// SQL: SELECT nome, status FROM produtos WHERE status <> 'ativo';
db.produtos.find({ status: { $ne: "ativo" } }, { _id: 0, nome: 1, status: 1 });

// R04 · Avaliações que ainda não foram aprovadas
// SQL: SELECT * FROM avaliacoes WHERE status <> 'aprovada';
db.avaliacoes.find({ status: { $ne: "aprovada" } }, { _id: 1, produto_id: 1, nota: 1, status: 1 });

// ---------------------------------------------------------------------
//  $gt · maior que
// ---------------------------------------------------------------------
// R05 · Produtos acima de R$ 1.000, do mais caro para o mais barato
// SQL: SELECT nome, preco FROM produtos WHERE preco > 1000 ORDER BY preco DESC;
db.produtos.find({ preco: { $gt: NumberDecimal("1000") } }, { _id: 0, nome: 1, preco: 1 }).sort({ preco: -1 });

// R06 · Pedidos com valor total acima de R$ 3.000
// SQL: SELECT * FROM pedidos WHERE valor_total > 3000;
db.pedidos.find({ valor_total: { $gt: NumberDecimal("3000") } }, { _id: 1, cliente_id: 1, status: 1, valor_total: 1 });

// ---------------------------------------------------------------------
//  $gte · maior ou igual a
// ---------------------------------------------------------------------
// R07 · Avaliações positivas (nota 4 ou 5)
// SQL: SELECT produto_id, nota, comentario FROM avaliacoes WHERE nota >= 4;
db.avaliacoes.find({ nota: { $gte: 4 } }, { _id: 0, produto_id: 1, nota: 1, comentario: 1 });

// R08 · SKUs com estoque confortável (20 unidades ou mais)
// SQL: SELECT sku, deposito, quantidade FROM estoque WHERE quantidade >= 20;
db.estoque.find({ quantidade: { $gte: 20 } }, { _id: 0, sku: 1, deposito: 1, quantidade: 1 }).sort({ quantidade: -1 });

// ---------------------------------------------------------------------
//  $lt · menor que
// ---------------------------------------------------------------------
// R09 · Produtos abaixo de R$ 100
// SQL: SELECT nome, preco FROM produtos WHERE preco < 100;
db.produtos.find({ preco: { $lt: NumberDecimal("100") } }, { _id: 0, nome: 1, preco: 1 });

// R10 · Alerta de reposição: SKUs com menos de 5 unidades
// SQL: SELECT sku, deposito, quantidade FROM estoque WHERE quantidade < 5;
db.estoque.find({ quantidade: { $lt: 5 } }, { _id: 0, sku: 1, deposito: 1, quantidade: 1 }).sort({ quantidade: 1 });

// ---------------------------------------------------------------------
//  $lte · menor ou igual a
// ---------------------------------------------------------------------
// R11 · Avaliações negativas (nota 2 ou menor)
// SQL: SELECT produto_id, nota, comentario FROM avaliacoes WHERE nota <= 2;
db.avaliacoes.find({ nota: { $lte: 2 } }, { _id: 0, produto_id: 1, nota: 1, comentario: 1 });

// R12 · Pedidos feitos até 15/09/2026
// SQL: SELECT * FROM pedidos WHERE data_pedido <= '2026-09-15 23:59:59';
db.pedidos.find({ data_pedido: { $lte: ISODate("2026-09-15T23:59:59Z") } }, { _id: 1, data_pedido: 1, status: 1 });

// ---------------------------------------------------------------------
//  $in · está na lista
// ---------------------------------------------------------------------
// R13 · Produtos de eletrônicos (Notebooks ou Smartphones)
// SQL: SELECT nome, categoria_id, preco FROM produtos WHERE categoria_id IN (4, 5);
db.produtos.find({ categoria_id: { $in: [4, 5] } }, { _id: 0, nome: 1, categoria_id: 1, preco: 1 });

// R14 · Pedidos em andamento (pagos ou enviados)
// SQL: SELECT * FROM pedidos WHERE status IN ('pago', 'enviado');
db.pedidos.find({ status: { $in: ["pago", "enviado"] } }, { _id: 1, cliente_id: 1, status: 1 });

// ---------------------------------------------------------------------
//  $nin · não está na lista
// ---------------------------------------------------------------------
// R15 · Pagamentos que não foram feitos por pix nem boleto (somente cartão)
// SQL: SELECT * FROM pagamentos WHERE metodo NOT IN ('pix', 'boleto');
db.pagamentos.find({ metodo: { $nin: ["pix", "boleto"] } }, { _id: 0, pedido_id: 1, metodo: 1, parcelas: 1, valor: 1 });

// R16 · Clientes fora do eixo SP-RJ
// SQL: SELECT nome, cidade, uf FROM clientes WHERE uf NOT IN ('SP', 'RJ');
db.clientes.find({ "endereco.uf": { $nin: ["SP", "RJ"] } }, { _id: 0, nome: 1, endereco: 1 });

// ---------------------------------------------------------------------
//  $and · todas as condições
// ---------------------------------------------------------------------
// R17 · Produtos ativos com preço entre R$ 200 e R$ 2.000
// SQL: SELECT nome, preco FROM produtos WHERE status = 'ativo' AND preco >= 200 AND preco <= 2000;
db.produtos.find(
  { $and: [ { status: "ativo" },
            { preco: { $gte: NumberDecimal("200") } },
            { preco: { $lte: NumberDecimal("2000") } } ] },
  { _id: 0, nome: 1, preco: 1 }
);

// R18 · Pagamentos no cartão de crédito parcelados em 3 vezes ou mais
// SQL: SELECT * FROM pagamentos WHERE metodo = 'cartao_credito' AND parcelas >= 3;
db.pagamentos.find(
  { $and: [ { metodo: "cartao_credito" }, { parcelas: { $gte: 3 } } ] },
  { _id: 0, pedido_id: 1, parcelas: 1, valor: 1, status: 1 }
);

// ---------------------------------------------------------------------
//  $or · pelo menos uma condição
// ---------------------------------------------------------------------
// R19 · Produtos nos extremos de preço (abaixo de R$ 100 ou acima de R$ 5.000)
// SQL: SELECT nome, preco FROM produtos WHERE preco < 100 OR preco > 5000;
db.produtos.find(
  { $or: [ { preco: { $lt: NumberDecimal("100") } }, { preco: { $gt: NumberDecimal("5000") } } ] },
  { _id: 0, nome: 1, preco: 1 }
);

// R20 · Pagamentos que exigem atenção (pendentes ou estornados)
// SQL: SELECT * FROM pagamentos WHERE status = 'pendente' OR status = 'estornado';
db.pagamentos.find(
  { $or: [ { status: "pendente" }, { status: "estornado" } ] },
  { _id: 0, pedido_id: 1, metodo: 1, valor: 1, status: 1 }
);

// ---------------------------------------------------------------------
//  $nor · nenhuma das condições
// ---------------------------------------------------------------------
// R21 · Produtos que não são camisetas e não custam mais de R$ 3.000
// SQL: SELECT nome, preco FROM produtos WHERE NOT (categoria_id = 2 OR preco > 3000);
db.produtos.find(
  { $nor: [ { categoria_id: 2 }, { preco: { $gt: NumberDecimal("3000") } } ] },
  { _id: 0, nome: 1, preco: 1 }
);

// R22 · Pedidos válidos: nem cancelados nem aguardando pagamento
// SQL: SELECT * FROM pedidos WHERE NOT (status = 'cancelado' OR status = 'aguardando_pagamento');
db.pedidos.find(
  { $nor: [ { status: "cancelado" }, { status: "aguardando_pagamento" } ] },
  { _id: 1, status: 1, valor_total: 1 }
);

// ---------------------------------------------------------------------
//  $not · nega a expressão
// ---------------------------------------------------------------------
// R23 · Produtos cujo preço NÃO é maior que R$ 500
// SQL: SELECT nome, preco FROM produtos WHERE NOT (preco > 500);
db.produtos.find({ preco: { $not: { $gt: NumberDecimal("500") } } }, { _id: 0, nome: 1, preco: 1 });

// R24 · Variantes que NÃO são de camisetas (SKU não começa com "CAM-")
// SQL: SELECT sku FROM variantes WHERE sku NOT LIKE 'CAM-%';
db.variantes.find({ sku: { $not: /^CAM-/ } }, { _id: 0, sku: 1, disponivel: 1 });

// ---------------------------------------------------------------------
//  $exists · campo existe ou não
// ---------------------------------------------------------------------
// R25 · Variantes que possuem aro (somente anéis)
// SQL: SELECT sku, aro FROM variantes WHERE aro IS NOT NULL;
db.variantes.find({ aro: { $exists: true } }, { _id: 0, sku: 1, aro: 1, disponivel: 1 });

// R26 · Categorias raiz (sem categoria pai)
// SQL: SELECT slug, nome FROM categorias WHERE categoria_pai_id IS NULL;
db.categorias.find({ categoria_pai_id: { $exists: false } }, { _id: 0, slug: 1, nome: 1 });

// ---------------------------------------------------------------------
//  $type · tipo BSON do campo
// ---------------------------------------------------------------------
// R27 · Atributos com valor numérico (servem para filtros por faixa)
// SQL: não há equivalente direto, no SQL a coluna valor era VARCHAR para todos
db.atributos_produto.find({ valor: { $type: "number" } }, { _id: 0, produto_id: 1, chave: 1, valor: 1, unidade: 1 });

// R28 · Atributos com valor textual
// SQL: não há equivalente direto, no SQL a coluna valor era VARCHAR para todos
db.atributos_produto.find({ valor: { $type: "string" } }, { _id: 0, produto_id: 1, chave: 1, valor: 1 });

// ---------------------------------------------------------------------
//  $regex · expressão regular
// ---------------------------------------------------------------------
// R29 · Produtos com "camiseta" no nome (sem diferenciar maiúsculas)
// SQL: SELECT nome FROM produtos WHERE LOWER(nome) LIKE '%camiseta%';
db.produtos.find({ nome: { $regex: "camiseta", $options: "i" } }, { _id: 0, nome: 1, status: 1 });

// R30 · Clientes com e-mail de domínio .com.br
// SQL: SELECT nome, email FROM clientes WHERE email LIKE '%.com.br';
db.clientes.find({ email: { $regex: /\.com\.br$/ } }, { _id: 0, nome: 1, email: 1 });

// ---------------------------------------------------------------------
//  $all · array contém todos os valores
// ---------------------------------------------------------------------
// R31 · Produtos marcados ao mesmo tempo como "algodao" e "unissex"
// SQL: exigiria uma tabela produto_tags e GROUP BY ... HAVING COUNT(*) = 2
db.produtos.find({ tags: { $all: ["algodao", "unissex"] } }, { _id: 0, nome: 1, tags: 1 });

// R32 · Sugestões de presente em prata
// SQL: exigiria uma tabela produto_tags e GROUP BY ... HAVING COUNT(*) = 2
db.produtos.find({ tags: { $all: ["presente", "prata"] } }, { _id: 0, nome: 1, tags: 1 });

// ---------------------------------------------------------------------
//  $elemMatch · um mesmo elemento do array atende a todas as condições
// ---------------------------------------------------------------------
// R33 · Clientes com celular de DDD 11
// SQL: SELECT c.nome FROM clientes c WHERE c.telefone_celular LIKE '11 %';
db.clientes.find(
  { telefones: { $elemMatch: { tipo: "celular", numero: /^11 / } } },
  { _id: 0, nome: 1, telefones: 1 }
);

// R34 · Clientes com telefone fixo fora do DDD 11
// SQL: SELECT nome FROM clientes WHERE telefone_fixo IS NOT NULL AND telefone_fixo NOT LIKE '11 %';
db.clientes.find(
  { telefones: { $elemMatch: { tipo: "fixo", numero: { $not: /^11 / } } } },
  { _id: 0, nome: 1, telefones: 1 }
);

// ---------------------------------------------------------------------
//  $size · quantidade de elementos do array
// ---------------------------------------------------------------------
// R35 · Produtos com exatamente 3 tags
// SQL: exigiria uma tabela produto_tags e GROUP BY produto_id HAVING COUNT(*) = 3
db.produtos.find({ tags: { $size: 3 } }, { _id: 0, nome: 1, tags: 1 });

// R36 · Clientes com dois telefones cadastrados
// SQL: SELECT nome FROM clientes WHERE telefone_celular IS NOT NULL AND telefone_fixo IS NOT NULL;
db.clientes.find({ telefones: { $size: 2 } }, { _id: 0, nome: 1, telefones: 1 });
