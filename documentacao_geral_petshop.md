# Sistema de Gerenciamento de Petshop

Documento de requisitos e arquitetura de microsserviços (C# / .NET)

## 1. Introdução e Problema do Negócio

### 1.1 Contexto

Petshops reúnem atividades de naturezas diferentes: atendimento a animais (banho, tosa, consulta veterinária), agenda de profissionais, venda de produtos, controle de estoque, compra de fornecedores e controle financeiro. Em estabelecimentos pequenos, essas informações costumam ficar espalhadas em planilhas, cadernos e mensagens.

### 1.2 Problema

- Dados de tutores e animais dispersos e duplicados.
- Conflitos de horário: dois atendimentos no mesmo horário para o mesmo profissional ou para o mesmo animal.
- Falta de visibilidade sobre as quantidades disponíveis em estoque.
- Dificuldade de relacionar compras e vendas com a movimentação financeira.
- Ausência de controle sobre quem pode executar cada operação.

### 1.3 Propósito do sistema

Desenvolver um sistema de gerenciamento para petshops que facilite e centralize o controle de tutores, animais, serviços, agendamentos, profissionais, estoque, compras, vendas e financeiro, tornando os processos administrativos e operacionais mais organizados.

## 2. Escopo do Produto e Regras de Negócio

### 2.1 Perfis de usuário

| Perfil | Responsabilidades |
| --- | --- |
| Administrador | Gerenciamento geral e controle de usuários e perfis; cadastro de serviços e de profissionais (incluindo disponibilidade). Pode executar tudo que os demais perfis executam. |
| Atendente | Cadastro e gerenciamento de tutores e animais; agendamentos; vendas. Consulta o catálogo de serviços. |
| Funcionário | Acompanhamento dos serviços e atualização do status dos atendimentos; estoque, compras e financeiro. |

### 2.2 Escopo

#### Dentro do escopo

- Usuários, perfis e autenticação.
- Tutores e animais.
- Catálogo de serviços (com preço e duração).
- Profissionais, disponibilidade semanal e serviços que executam.
- Agendamentos com controle de status e de conflitos de horário.
- Estoque de produtos, com movimentações manuais, de compras e de vendas.
- Fornecedores, compras e vendas de produtos, com receitas, despesas e situação de pagamentos.

#### Fora do escopo

- Detalhamento de formas de pagamento, parcelamento e integração com meios de pagamento reais (o preço é apenas um atributo de serviços e produtos).
- Consumo automático de estoque por serviços (a quantidade usada varia a cada atendimento, então a baixa é manual).
- Recebimento parcial de compras.

### 2.4 Regras de negócio

#### Agendamentos

| ID | Regra |
| --- | --- |
| RN01 | Um agendamento ocupa o intervalo [início, início + duração do serviço). A duração é copiada do serviço no momento da criação. |
| RN02 | Um profissional/animal não pode ter dois agendamentos ativos com intervalos sobrepostos. |
| RN03 | O agendamento deve estar dentro da disponibilidade do profissional. |
| RN04 | Animal, serviço e profissional devem estar ativos para novos agendamentos. |
| RN05 | O agendamento guarda o preço e a duração do serviço no momento da criação; mudanças posteriores no serviço não o alteram. |
| RN06 | Somente agendamentos com status Agendado podem ser remarcados ou cancelados. |
| RN07 | O Funcionário registra os status Agendado, Em andamento e Concluído/Cancelado/Faltou. |

#### Pagamento de serviços

| ID | Regra |
| --- | --- |
| RN08 | O pagamento ocorre após o atendimento. Agendar nunca gera lançamento financeiro. |
| RN09 | O agendamento não movimenta estoque. O consumo de produtos nos serviços é lançado manualmente. |

#### Compras/Vendas

| ID | Regra |
| --- | --- |
| RN10 | Status da compra: Pendente → Confirmada ou Cancelada. |
| RN11 | Somente a confirmação gera efeitos no estoque. (Compra pendente ou cancelada não altera) |
| RN12 | A confirmação deve ser atômica do ponto de vista do usuário: ou ambos os efeitos ocorrem, ou a compra permanece Pendente. |
| RN13 | A venda só é registrada se houver quantidade suficiente em estoque para todos os itens. |
| RN14 | Registrar a venda gera saída no estoque. |

#### Estoque

| ID | Regra |
| --- | --- |
| RN16 | A quantidade em estoque nunca pode ficar negativa. |
| RN17 | A origem de uma movimentação é Manual, Compra ou Venda. |

#### Tutores e Animais

| ID | Regra |
| --- | --- |
| RN19 | Todo animal pertence a exatamente um tutor, definido no cadastro do animal; não existe animal sem tutor. |
| RN20 | Tutores e animais não são excluídos, são inativados; o histórico de agendamentos e vendas é preservado. |
| RN21 | Não é possível cadastrar nem reativar um animal vinculado a um tutor inativo. Ao inativar um tutor, seus animais também são inativados. |
| RN22 | A inativação de um tutor ou de um animal não altera agendamentos já criados. |

#### Serviços do profissional

| ID | Regra |
| --- | --- |
| RN23 | Todo serviço possui preço (maior ou igual a zero) e duração em minutos (maior que zero), ambos obrigatórios. |
| RN26 | Um profissional só pode ser agendado para os serviços vinculados a ele. |
| RN27 | A disponibilidade do profissional é semanal e recorrente: cada faixa define dia da semana, hora inicial e hora final (maior que a inicial), e faixas do mesmo dia não se sobrepõem. |
| RN29 | A inativação de um serviço ou de um profissional e a alteração de disponibilidade não alteram agendamentos já criados. |

## 3. Requisitos Funcionais e Não Funcionais

### 3.1 Requisitos funcionais

#### Usuários

| ID | Requisito |
| --- | --- |
| RF01 | Gerenciar usuários (cadastrar, editar, inativar). |
| RF02 | Atribuir perfil ao usuário. |
| RF03 | Autenticar usuário e emitir token de acesso contendo o perfil. |

#### Tutores e Animais

| ID | Requisito |
| --- | --- |
| RF04 | Gerenciar tutores. |
| RF05 | Gerenciar animais. |
| RF06 | Vincular animal a um tutor (todo animal possui exatamente um tutor). |

#### Agendamentos (inclui catálogo de serviços e profissionais)

| ID | Requisito |
| --- | --- |
| RF07 | Gerenciar serviços oferecidos (nome, descrição, preço e duração em minutos). |
| RF08 | Gerenciar profissionais. |
| RF09 | Definir a disponibilidade semanal do profissional. |
| RF10 | Definir quais serviços o profissional executa. |
| RF11 | Gerenciar agendamentos |

#### Estoque

| ID | Requisito |
| --- | --- |
| RF12 | Gerenciar produtos (incluindo preço de venda). |
| RF13 | Registrar entradas e saídas manuais de produtos. |
| RF14 | Consultar quantidades disponíveis e histórico de movimentações. |

#### Comercial (inclui financeiro)

| ID | Requisito |
| --- | --- |
| RF15 | Gerenciar fornecedores. |
| RF16 | Gerenciar compras/vendas. |
| RF17 | Registrar receitas/despesas. |
| RF18 | Consultar movimentações financeiras. |
| RF19 | Controlar a situação dos pagamentos (pendente e realizado). |

### 3.2 Requisitos não funcionais

| ID | Requisito |
| --- | --- |
| RNF01 | Cada serviço é executado, testado e implantado de forma independente, com Dockerfile e pipeline próprios. |
| RNF02 | Cada serviço possui banco de dados isolado; o acesso a dados de outro serviço ocorre apenas via API. |
| RNF03 | Autenticação por token validado localmente por cada serviço; autorização por perfil. |
| RNF04 | A falha de um serviço não derruba os demais; chamadas entre serviços possuem timeout e retornam erro explícito (503) quando a dependência está indisponível. |
| RNF05 | Operações que envolvem vários serviços são idempotentes, ou seja, seguras para repetição. |
| RNF07 | Cada serviço expõe documentação OpenAPI e um endpoint /health. |
| RNF08 | Configurações e segredos são fornecidos por variáveis de ambiente, nunca armazenados no repositório. |

## 4. Identificação dos Microsserviços e suas Relações (Macro)

### 4.1 Microsserviços

O sistema é composto por cinco microsserviços, cada um em seu próprio repositório (multirepo). Um repositório adicional de infraestrutura contém o docker-compose que sobe todos os serviços e bancos para execução integrada local.

| Serviço | Responsabilidade | Papel |
| --- | --- | --- |
| Usuários | Usuários, perfis e emissão de token | Folha (plataforma) |
| Tutores e Animais | Cadastro de tutores e animais e seu vínculo | Folha |
| Agendamentos | Serviços (preço e duração), profissionais e disponibilidade, agendamentos, status e controle de conflitos | Consumidor |
| Estoque | Produtos, quantidades e movimentações | Folha |
| Comercial | Fornecedores, compras, vendas, receitas, despesas e situação de pagamentos | Orquestrador |

### 4.2 Relações entre os serviços

| Origem | Destino | Finalidade |
| --- | --- | --- |
| Agendamentos | Tutores e Animais | Validar animal e obter dados do tutor (leitura) |
| Comercial | Estoque | Entrada na compra confirmada; saída na venda; consulta de produto e preço |
| Comercial | Tutores e Animais | Validar tutor opcional da venda (leitura) |
| Todos os serviços | Usuários (via token) | Validação local do token, sem chamada a cada requisição |

## 5. Arquitetura Geral de Cada Serviço (Micro)

### 5.1 Estrutura padrão

Todos os serviços partem do mesmo template e seguem a mesma organização interna, deliberadamente simples:

- Controllers: recebem as requisições HTTP, validam a entrada e delegam ao Service.
- Servicos: regras de negócio; acessam o contexto do Entity Framework Core diretamente quando apropriado.
- DTO: objetos de entrada e saída da API, separados das entidades (entidades nunca são expostas).
- Entities: classes persistidas no banco.
- Data: DataContext do Entity Framework Core e migrations.
- Infra: clientes HTTP para outros serviços (apenas nos serviços que têm dependências).
