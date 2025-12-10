# Relatório Técnico – Análise de Segurança do Servidor Web ESP32

## 1. Introdução
Este relatório apresenta a análise estática e dinâmica do código de um servidor web simples implementado no ESP32, baseado no exemplo do Random Nerd Tutorials.  
O objetivo é identificar vulnerabilidades, descrever possíveis ataques, estimar risco e consolidar os resultados em uma tabela técnica.

## 2. Análise Estática do Código

### 2.1 Vulnerabilidades Identificadas

#### **V1 — Ausência de Autenticação**
O código não possui nenhum tipo de autenticação ou controle de acesso.  
Qualquer dispositivo na mesma rede pode acionar as rotas `/26/on`, `/26/off`, `/27/on`, `/27/off`.

#### **V2 — Comunicação sem Criptografia (HTTP)**
O ESP32 usa HTTP puro, sem TLS.  
Isso permite ataques de sniffing e manipulação de tráfego.

#### **V3 — Falta de Sanitização do Header**
A variável `header` é preenchida diretamente com o conteúdo enviado pelo cliente.  
Isso abre espaço para:
- injeção de payloads inesperados,  
- buffer growth (memória aumenta indefinidamente),  
- ataques de request smuggling ou malformed HTTP.

#### **V4 — Exposição de Informações Sensíveis**
O ESP32 imprime na serial:
- SSID usado,
- IP local,
- todo o conteúdo do header HTTP.

Em ambiente físico compartilhado, isso expõe dados importantes.

#### **V5 — Falta de Limitação de Requisições (Rate Limiting)**
O servidor aceita conexões ilimitadas, sem mitigação contra:
- DoS,
- conexões repetidas,
- floods de requests.

## 3. Descrição de Dois Ataques

### ### **Ataque 1 – Acesso Não Autorizado ao Controle dos GPIOs**
(Relação com V1: falta de autenticação)

#### **Passo a passo**
1. Atacante conecta-se à mesma rede Wi-Fi do ESP32.  
2. Descobre o IP do dispositivo via varredura Nmap ou roteador, por exemplo.  
3. Acessa no navegador:  
   `http://<ESP_IP>/26/on`  
4. Controla GPIOs remotamente sem qualquer validação.

#### **Probabilidade**
**Alta**  
Já que ambientes de laboratório, residenciais e makers raramente têm segmentação de rede ou senhas fortes.

#### **Impacto**
**Médio a Alto**  
Dependendo do que está conectado ao GPIO pode causar danos físicos ou falhas operacionais.

#### **Risco (Probabilidade × Impacto)**
**Alto**

Em resumo, essa vulnerabilidade compromete diretamente a integridade e a disponibilidade do sistema, além de abrir caminho para ataques mais complexos. Por exemplo, possibilita sabotagem, a interrupção de serviços ou danos ao hardware conectado ao microcontrolador.  

### ### **Ataque 2 – Sniffing e Manipulação de Tráfego (MITM)**
(Relação com V2: ausência de criptografia)

#### **Passo a passo**
1. Atacante conectado à mesma rede ativa modo MITM com ferramentas como:  
   - Wireshark  
   - Bettercap  
   - ARP spoofing  
2. Intercepta os pacotes HTTP enviados ao ESP32.  
3. Lê claramente tráfego como:  
`GET /27/on HTTP/1.1`
4. Pode alterar o request e reenviar comandos automaticamente.

#### **Probabilidade**
**Média**  
Requer conhecimento técnico, mas é simples em redes Wi-Fi abertas ou corporativas sem isolamento.

#### **Impacto**
**Médio**  
Permite execução arbitrária de comandos.  
Com manipulação contínua, pode causar:
- acionamentos repetitivos,  
- sobreaquecimento,  
- comportamento inesperado dos dispositivos.

#### **Risco**
**Médio-Alto**

## 4. Consolidação das Análises

Resumo geral das vulnerabilidades e impactos:

- O sistema não tem controles de acesso.  
- O tráfego não é protegido.  
- O cabeçalho HTTP não é sanitizado.  
- O sistema imprime informações sensíveis na serial.  
- Não há proteção contra DoS.

Esses fatores fazem o servidor inadequado para uso em ambientes reais sem reforço de segurança.

## 5. Tabela Consolidada de Ataques (Ordem Decrescente de Risco)

| Título do Ataque                           | Probabilidade | Impacto | Risco |
|--------------------------------------------|--------------|---------|-------|
| Acesso Não Autorizado ao Controle de GPIOs | Alta         | Médio/Alto | **Alto** |
| Sniffing e Manipulação de Tráfego (MITM)   | Média        | Médio   | **Médio-Alto** |

## 6. Análise Dinâmica 

### **Montagem e Teste Prático**
Seguindo o tutorial, o ESP32 foi montado em protoboard e programado com o código fornecido.

#### **Teste de Ataque Executado**
**Ataque testado: Acesso não autorizado**

**Procedimento:**
1. ESP32 conectado ao Wi-Fi.  
2. Atacante conectado à mesma rede.  
3. Abrir navegador → `http://<IP_DO_ESP>`  
4. Pressionar botões *sem qualquer login*.  
5. GPIOs 26 e 27 responderam normalmente.

#### **Resultados Observados**
- Qualquer dispositivo na rede consegue controlar os atuadores.  
- Não há logs de origem do acesso.  
- Não existe mitigação contra repetição rápida de requests.  

[Link para o vídeo de ataque ao esp](https://youtube.com/shorts/E1-FKJS24k4)
