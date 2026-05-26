# Zézin - Sistema Automático de Triagem e Atendimento

O Zézin é um ecossistema de automação criado no **n8n** para operar como um assistente de suporte de Nível 1 diretamente no Discord. O objetivo é organizar chamados, definir prioridades de forma dinâmica, coletar dados estruturados e monitorar o tempo de resposta (SLA).

## 🛠️ Como Funciona a Arquitetura

O sistema é dividido em quatro fluxos independentes, garantindo estabilidade e escalabilidade sem sobrecarregar um único processo:

* **Zézin Construtor:** Monitora canais de suporte no Discord e, ao identificar uma solicitação, cria uma Thread isolada (sala de atendimento) garantindo que chamados não sejam duplicados.
* **Zezin Categorização:** Interage dentro da Thread pedindo para classificar a gravidade da demanda (P0, P1, P2, Priorização ou Informação). Após a resposta, gera um link dinâmico de formulário no Tally para coleta dos detalhes técnicos e identidade do usuário.
* **Zezin Triagem:** Recebe a carga de dados preenchidos no formulário via Webhook (em tempo real), atualiza o status do banco de dados (Google Sheets) e notifica a equipe técnica na respectiva sala.
* **Zézin Analises:** Atua como um auditor contínuo do ecossistema. Monitora ativamente o tempo entre a criação da sala e o preenchimento dos dados, disparando alertas de cobrança caso o usuário deixe o chamado abandonado.

---

## 🚀 Instalação e Uso

### 1. Pré-requisitos

* **Docker** e **Docker Compose** instalados no servidor ou ambiente local.
* Um bot configurado no **Discord Developer Portal**.
* Conta no **Google Cloud** com a API do Google Sheets habilitada.
* Uma conta no **Tally.so** para gerenciar o formulário de coleta.
* Uma ferramenta de túnel (Zrok, Cloudflare Tunnels ou Ngrok) configurada, caso o n8n esteja rodando localmente (fora de uma VPS).

### 2. Subindo a Infraestrutura

Clone o repositório e inicie a estrutura utilizando o Docker:

```bash
git clone https://github.com/SEU-USUARIO/NOME-DO-REPOSITORIO.git
cd NOME-DO-REPOSITORIO
docker compose up -d
```

Acesse o painel do n8n através de `http://localhost:5678`.

### 3. Importando os Fluxos

1. Dentro do n8n, acesse a aba **Workflows**.
2. Clique no menu superior direito e escolha a opção de importar via arquivo (*Import from File*).
3. Selecione e importe os quatro arquivos `.json` presentes neste repositório.

### 4. Configurando Credenciais e Nós

* Adicione as credenciais do **Discord Bot API** utilizando o Token oficial do seu bot.
* Crie as credenciais do **Google Sheets OAuth2 API** para permitir a leitura e escrita.
* Abra os nós do Google Sheets presentes nos fluxos e garanta que estão apontando para a sua planilha oficial.
* Nos nós do Discord, substitua os `IDs` dos canais e servidores pelos `IDs` do seu próprio servidor.

### 5. Configurando o Webhook

1. Abra o fluxo **Zezin Triagem**.
2. Acesse o nó inicial de Webhook e copie a URL de Produção (*Production URL*).
3. Cole a URL final nas configurações de integração (Webhooks) do seu formulário no Tally.so.

### 6. Colocando no Ar

Com as conexões estabelecidas, clique no botão **Publish** localizado no canto superior direito de cada um dos quatro fluxos.

---

## 📈 Roteiro de Escalabilidade (Futuro do Projeto)

Para transformar este MVP funcional em uma infraestrutura pronta para suportar milhares de chamados simultâneos sem gargalos de API ou lentidão, o projeto prevê a seguinte evolução técnica:

### 1. Migração de Armazenamento: Google Sheets ➔ Banco de Dados Relacional (PostgreSQL / Supabase)

* **Porquê:** O Google Sheets possui limites severos de requisições por minuto (Rate Limits / Erro 429). A migração para um banco de dados como o PostgreSQL elimina travas de leitura/escrita.
* **Performance:** A busca por `thread_id` passa a ser indexada (Chave Primária), reduzindo o tempo de consulta a frações de milissegundos, independentemente do volume de registros acumulados.
* **Integridade:** Garante conformidade ACID, evitando perda de dados caso múltiplos chamados sejam criados ou atualizados no exato mesmo milissegundo (Race Conditions).

### 2. Mudança de Arquitetura de Gatilhos: Polling ➔ Push (Event-Driven)

* **Porquê:** Atualmente, os fluxos utilizam loops de tempo curto (*Schedule Trigger*) para varrer a API do Discord. Isso gera um desperdício maciço de processamento em momentos ociosos da comunidade.
* **Evolução:** Substituir os cronômetros pelo nó nativo **Discord Trigger** ou pela API de Interações do Discord. O fluxo permanecerá dormente e só consumirá recursos da CPU quando um evento real acontecer no servidor.

### 3. Camada de Visualização e Métricas Avançadas (Metabase / Grafana Open Source)

* **Porquê:** Com a saída do Google Sheets, a equipe operacional perde a interface visual direta para acompanhamento e leitura dos chamados.
* **Evolução:** Adicionar um contêiner Docker com **Metabase** ou **Grafana (OSS)** plugado diretamente ao PostgreSQL. Isso permite criar painéis de controle em tempo real para monitorar a fila de chamados pendentes, médias de tempo de atendimento (SLA) e volumetria por categoria de erro, rodando 100% local e gratuito.

### 4. Modularização por Sub-Workflows (`Execute Workflow`)

* **Porquê:** Centralizar regras complexas de triagem e caminhos condicionais (P0, P1, P2, etc.) dentro de um único nó Switch dificulta a manutenção do código e os testes individuais de novos recursos.
* **Evolução:** Isolar os blocos de ação utilizando o nó `Execute Workflow`. O fluxo principal atuará estritamente como um roteador inteligente de tráfego, enquanto fluxos filhos processam de forma isolada as regras de cada prioridade, blindando o ecossistema contra quebras generalizadas.

### 5. Filas de Processamento para Alta Disponibilidade (Redis / RabbitMQ)

* **Porquê:** Em picos de grande tráfego ou em momentos de instabilidade em APIs externas, uma enxurrada de preenchimentos simultâneos no Tally pode sobrecarregar a memória do contêiner do n8n.
* **Evolução:** Implementar uma fila de mensageria intermediária. O webhook do Tally apenas deposita o dado bruto na fila instantaneamente, e o n8n consome as mensagens da fila em um ritmo cadenciado e controlado, garantindo resiliência total contra perda de chamados.