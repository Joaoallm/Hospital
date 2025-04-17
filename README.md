# 📋 Projeto de Banco de Dados - Sistema Hospitalar

Este projeto tem como objetivo criar, popular e consultar um banco de dados para o sistema clínico de um hospital. A proposta abrange desde o levantamento de requisitos até consultas avançadas.

---

## 🧩 Parte 1 - Mãos à Obra

### 📝 Descrição
Desenvolvimento do modelo de dados para controle de consultas médicas, incluindo cadastro de médicos, pacientes, convênios e receituários. 

### 🔍 Requisitos Identificados
- Cadastro de médicos com tipo (Generalista, Especialista, Residente) e especialidades.
- Cadastro de pacientes com dados pessoais, documentos e convênio.
- Registro de consultas com data, hora, médico, paciente, valor, convênio e especialidade.
- Registro de receituários com medicamentos, quantidades e instruções.
- Migração de registros físicos para o sistema digital.

---

## 🏥 Parte 2 - Não Era Exatamente Assim

### 📝 Expansão dos Requisitos
- Controle de internações com datas, procedimentos e quartos.
- Tipos de quarto (Apartamento, Quarto Duplo, Enfermaria) com valor diário.
- Cadastro de enfermeiros com nome, CPF e COREN.
- Vínculo entre internação, paciente, médico e enfermeiros.

---

## 🧪 Parte 3 - Jogando nas Regras

### 🎯 Objetivo
Popular as coleções do banco com dados realistas e coerentes com os requisitos.

### ✅ Itens obrigatórios
- 12 médicos de diferentes especialidades.
- 7 especialidades no total.
- 15 pacientes.
- 20 consultas (algumas com receituários contendo 2 ou mais medicamentos).
- 7 internações (com pacientes internando mais de uma vez).
- 3 tipos de quarto cadastrados.
- 10 enfermeiros.
- Associação correta entre internações, médicos, pacientes, enfermeiros e quartos.

---

## 🛠️ Parte 4 - Inserindo Dados

### ✏️ Alterações no Banco
- Adição do campo `em_atividade` nos documentos de médicos.
  R: Campo já criado em meio a criação do banco de dados, chamado Status
  
  Q:
  ```js
  db.medicos.find(
    { _id: "1" }, // Filtro para localizar o médico com id 1
    { status: 1, _id: 0 } // Projeção: Mostra apenas o campo "status", ocultando o "_id"
  ```
  ---
- Atualização de pelo menos dois médicos como inativos.
  Q:
  ```js
  db.Medicos.updateOne({ _id: "1" }, { $set: { status: 0 } });
  db.Medicos.updateOne({ _id: "2" }, { $set: { status: 0 } });

  ```
- Atualização dos demais como ativos.
 Q:
  ```js
  db.Medicos.updateMany({}, { $set: { status: 1 } });
  ```
---

## 🔎 Parte 5 - Consultas Para Que Te Quero

### 📌 Consultas Requeridas
- Todas as consultas de 2020 com valor médio e filtro por convênio.
  R: 150

  Q:
  ```js
  db.consultas.aggregate([
  {
    $match: {
      data_hora: { $gte: "2020-01-01", $lte: "2020-12-31" }
    }
  },
  {
    $group: {
      _id: null,
      valorMedio: { $avg: "$custo" }
    }
  }
  ])
  ```
  ---
- Internações com alta após a data prevista.
  R: 0

  Q:
  ```js
  db.internacoes.find({
  $expr: {
    $gt: ["$data_efetiva_alta", "$data_prevista_alta"]
  }
  })
  ```
  ---
- Receituário da primeira consulta com prescriçã
  Q:
  ```js
  db.consultas.find(
  { receita_medicamentos: { $exists: true, $ne: [] } }
  )
  .sort({ data_hora: 1 })
  .limit(1)
  ```
  ---
- Consultas com maior e menor valor (sem convênio).
  Q:
  ```js
  db.consultas.find(
  { convenio: { $eq: null } } 
  ).sort({ custo: -1 }).limit(1); 
  ```
  ---
- Internações com total calculado (valor da diária × número de dias).
  Q:
  ```js
  db.internações.aggregate([
  {
    $addFields: {
      data_entrada_convertida: { $dateFromString: { dateString: "$data_entrada" } },
      data_alta_convertida: { $dateFromString: { dateString: "$data_efetiva_alta" } }
    }
  },
  {
    $addFields: {
      diasInternacao: {
        $dateDiff: {
          startDate: "$data_entrada_convertida",
          endDate: "$data_alta_convertida",
          unit: "day"
        }
      }
    }
  },
  {
    $addFields: {
      total_internacao: { $multiply: ["$valor_diaria", "$diasInternacao"] }
    }
  },
  {

  ```
  ---
- Internações em apartamentos (data, procedimento, nº do quarto).
   Q:
  ```js
  db.internações.aggregate([
  {
    $match: { tipo_quarto: "Apartamento" } // Filtra apenas internações em apartamentos
  },
  {
    $project: {
      _id: 0,  
      data_entrada: 1, 
      procedimentos_realizados: 1, 
      quarto_id: 1 
    }
  }
  ]);
  ```
  ---
- Consultas de pacientes menores de 18 anos com especialidades que **não** sejam pediatria.
  R: 0

  Q:
  ```js
  db.consultas.aggregate([
  {
    $match: {
      especialidade_responsavel: { $ne: "Pediatra" } 
    }
  },
  {
    $addFields: {
      idade_paciente: {
        $dateDiff: {
          startDate: "$data_nascimento", 
          endDate: "$data_consulta",     
          unit: "year"                   
        }
      }
    }
  },
  {
    $match: {
      idade_paciente: { $lt: 18 } // Filtra apenas pacientes menores de 18 anos
    }
  },
  {
    $project: {
      nome_paciente: 1,            
      data_consulta: 1,            
      especialidade_responsavel: 1 
    }
  }
  ]);
  ```
  ---
- Internações realizadas por médicos gastroenterologistas em **enfermaria**.
  R: 1

   Q:
  ```js
  db.internacoes.aggregate([
  {
    $match: {
      especialidade_responsavel: "Gastroenterologista", // Filtra médicos com especialidade Gastroenterologista
      tipo_quarto: "Enfermaria" // Filtra internações em quartos do tipo Enfermaria
    }
  },
  {
    $project: {
      _id: 0, // Exclui o campo _id do resultado
      nome_paciente: 1, // Nome do paciente internado
      nome_medico_responsavel: 1, // Nome do médico responsável
      data_entrada: 1, // Data de entrada do paciente
      procedimentos_realizados: 1, // Procedimentos realizados durante a internação
      quarto_id: 1 // Número do quarto onde ocorreu a internação
    }
  }
  ]);
  ```
  ---
- Médicos com seus CRMs e quantidade de consultas realizadas.
   Q:
  ```js
  db.medicos.aggregate([
  {
    $project: {
      nome_medico: "$nome",        // Nome do médico
      CRM: "$documentos.CRM",      // CRM do médico
      consultas_realizadas: {
        $arrayElemAt: [
          { $split: ["$historico_consultas", " "] }, 1
        ]
      } // Extrai a quantidade de consultas realizadas do campo `historico_consultas`
    }
  }
  ]);
  ```
  ---
- Médicos com “Gabriel” no nome.
  R:1

 Q:
  ```js
  db.medicos.find({
nome:/Gabriel/
})
  ```
  ---
- Enfermeiros com mais de uma internação (nome, COREN, número de internações).
Q:
  ```js
  db.internações.aggregate([
  {
    $unwind: "$enfermeiros_responsaveis" // Desestrutura o array de enfermeiros responsáveis
  },
  {
    $group: {
      _id: "$enfermeiros_responsaveis", // Agrupa pelo identificador do enfermeiro (COREN)
      totalInternacoes: { $sum: 1 }    // Soma o número de internações por enfermeiro
    }
  },
  {
    $match: { totalInternacoes: { $gt: 1 } } // Filtra enfermeiros com mais de uma internação
  },
  {
    $lookup: {
      from: "enfermeiros",                // Junta com a coleção de enfermeiros
      localField: "_id",                  // Campo identificador do enfermeiro (COREN)
      foreignField: "documentos.coren",   // Campo correspondente na coleção de enfermeiros
      as: "detalhesEnfermeiro"            // Nome do array com informações do enfermeiro
    }
  },
  {
    $project: {
      coren: "$_id",                                     // COREN do enfermeiro
      nome: { $arrayElemAt: ["$detalhesEnfermeiro.nome", 0] }, // Nome do enfermeiro
      totalInternacoes: 1                               // Número de internações
    }
  }
  ]);
  ```
  ---
  ---

## 🧰 Tecnologias Utilizadas

- **Banco de Dados:** MongoDB
- **Scripts:** MongoDB Shell
- **Modelagem:** Documentos JSON

---


