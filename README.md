# Monitoramento de Câmeras Offline no Home Assistant

Este projeto cria uma automação no Home Assistant para monitoramento e notificação de câmeras de segurança que ficam offline. A automação utiliza o `notify`, mensagens no Telegram e notificações persistentes para informar sobre o estado das câmeras, garantindo que o usuário saiba imediatamente quando uma ou mais câmeras perdem a conexão.

## Descrição do Código

O código tem como objetivo:
- Detectar quando uma câmera passa para o estado "unavailable" (offline).
- Aguardar 1 minuto para garantir que a câmera permaneça offline antes de notificar.
- Enviar notificações no Telegram, no aplicativo Home Assistant e criar notificações persistentes na interface.
- Repetir notificações a cada 5 minutos enquanto as câmeras estiverem offline.
- Enviar notificações quando todas as câmeras voltarem ao estado online.

### Estrutura do Código

- **Variáveis de Configuração**: Permitem fácil ajuste dos parâmetros, como lista de câmeras monitoradas, `telegram_chat_id`, tempo de atraso inicial e período de verificação.
- **Gatilho (Trigger)**: Detecta qualquer câmera na lista de `entity_id` que fique offline.
- **Ações**:
  - Aguardar o atraso inicial.
  - Verificar se alguma câmera permanece offline após 1 minuto.
  - Enviar notificações detalhadas para o Home Assistant, Telegram e notificação persistente.
  - Repetir essas notificações a cada 5 minutos enquanto houver câmeras offline.
  - Notificar quando todas as câmeras voltarem ao estado online.
  
### Detalhamento das Ações

1. **Atraso Inicial**: Ação de `delay` para aguardar 1 minuto, garantindo que a câmera esteja realmente offline antes de notificar.
2. **Verificação**: Usa um template para verificar se uma ou mais câmeras estão offline há mais de 1 minuto.
3. **Notificações**: Envia mensagens detalhadas para `notify.notify`, Telegram e `persistent_notification.create`.
4. **Verificação Contínua**: Repetição da verificação e notificação a cada 5 minutos enquanto as câmeras estiverem offline.
5. **Notificação de Restauração**: Notifica quando todas as câmeras voltam ao estado online.

## Como Utilizar

1. **Pré-requisitos**: Ter o Home Assistant configurado com notificações via Telegram.
2. **Configuração**:
   - Substitua `telegram_chat_id` pelo seu chat ID.
   - Ajuste `cameras_para_monitorar` para incluir as câmeras de sua rede.
3. **Automação**: Copie o código para a configuração de automação do Home Assistant.

## Exemplo de Código

```yaml
# Nome e descrição
alias: Notificar e Verificar Câmeras Offline
description: >
  Notifica no Telegram, no aplicativo e na interface do Home Assistant quando câmeras
  ficam offline, com notificações periódicas enquanto as câmeras estiverem offline.

# Gatilho para ativar a automação quando qualquer câmera na lista ficar offline
trigger:
  - platform: state
    entity_id:
      - camera.camera1
      - camera.camera2
      - camera.camera3
      - camera.camera4
      - camera.camera5
      - camera.camera6
      - camera.camera7
      - camera.camera8
      - camera.camera0_video_doorbell
    to: unavailable

# Ações a serem executadas quando o gatilho é ativado
action:
  - delay: "{{ atraso_inicial }}" # Aguardar o tempo de atraso inicial configurado
  - condition: template
    value_template: >
      {% set cameras_offline = states | selectattr('entity_id', 'in', cameras_para_monitorar) | selectattr('state', 'equalto', 'unavailable') | list %}
      {% set cameras_offline_maior_1min = cameras_offline | selectattr('last_changed', 'lt', (now() - timedelta(minutes=1))) | list %}
      {{ cameras_offline_maior_1min | length > 0 }}
  - service: notify.notify
    data:
      message: >
        {% for camera in cameras_offline %}
        - {{ camera.entity_id.split('.')[1].replace('_', ' ') }} - Offline desde 
        {{ (camera.last_changed | as_datetime | as_local).strftime('%d/%m/%y às %H:%M:%S') }} 
        ({{ ((as_timestamp(now()) - as_timestamp(camera.last_changed)) / 60) | round(1) }} min) \n
        {% endfor %}
  # Outras ações: Telegram, notificações persistentes e verificações contínuas
variables:
  telegram_chat_id: "<seu_chat_id>"
  periodo_verificacao: "00:05:00"
  atraso_inicial: "00:01:10"
  cameras_para_monitorar:
    - camera.camera1
    - camera.camera2
    - camera.camera3
    - camera.camera4
    - camera.camera5
    - camera.camera6
    - camera.camera7
    - camera.camera8
    - camera.camera0_video_doorbell
mode: single
```


# Offline Camera Monitoring in Home Assistant

This project creates an automation in Home Assistant to monitor and notify when security cameras go offline. It uses `notify`, Telegram messages, and persistent notifications to alert the user as soon as one or more cameras lose connection.

## Code Description

The purpose of this code is to:
- Detect when a camera enters the "unavailable" state (offline).
- Wait 1 minute to ensure the camera remains offline before sending a notification.
- Send notifications to Telegram, the Home Assistant app, and create persistent notifications in the interface.
- Repeat notifications every 5 minutes as long as cameras are offline.
- Notify when all cameras are back online.

### Code Structure

- **Configuration Variables**: Allow easy adjustment of parameters, such as monitored cameras list, `telegram_chat_id`, initial delay time, and verification period.
- **Trigger**: Detects any camera in the `entity_id` list that goes offline.
- **Actions**:
  - Wait for the initial delay.
  - Check if any camera remains offline for more than 1 minute.
  - Send detailed notifications to Home Assistant, Telegram, and persistent notifications.
  - Repeat these notifications every 5 minutes as long as there are offline cameras.
  - Notify when all cameras return to the online state.

### Action Details

1. **Initial Delay**: `delay` action to wait 1 minute, ensuring the camera is really offline before notifying.
2. **Verification**: Uses a template to check if one or more cameras are offline for over 1 minute.
3. **Notifications**: Sends detailed messages to `notify.notify`, Telegram, and `persistent_notification.create`.
4. **Continuous Verification**: Repeats verification and notification every 5 minutes while cameras are offline.
5. **Restore Notification**: Notifies when all cameras are back online.

## Usage

1. **Prerequisites**: Have Home Assistant configured with Telegram notifications.
2. **Configuration**:
   - Replace `telegram_chat_id` with your chat ID.
   - Adjust `cameras_para_monitorar` to include the cameras in your network.
3. **Automation**: Copy the code into your Home Assistant automation configuration.

## Example Code

```yaml
# Name and Description
alias: Notify and Check Offline Cameras
description: >
  Notifies on Telegram, the Home Assistant app, and the interface when cameras
  go offline, with periodic notifications while cameras remain offline.

# Trigger to activate automation when any camera in the list goes offline
trigger:
  - platform: state
    entity_id:
      - camera.camera1
      - camera.camera2
      - camera.camera3
      - camera.camera4
      - camera.camera5
      - camera.camera6
      - camera.camera7
      - camera.camera8
      - camera.camera0_video_doorbell
    to: unavailable

# Actions to execute when the trigger is activated
action:
  - delay: "{{ atraso_inicial }}" # Wait for the configured initial delay
  - condition: template
    value_template: >
      {% set cameras_offline = states | selectattr('entity_id', 'in', cameras_para_monitorar) | selectattr('state', 'equalto', 'unavailable') | list %}
      {% set cameras_offline_maior_1min = cameras_offline | selectattr('last_changed', 'lt', (now() - timedelta(minutes=1))) | list %}
      {{ cameras_offline_maior_1min | length > 0 }}
  - service: notify.notify
    data:
      message: >
        {% for camera in cameras_offline %}
        - {{ camera.entity_id.split('.')[1].replace('_', ' ') }} - Offline since 
        {{ (camera.last_changed | as_datetime | as_local).strftime('%d/%m/%y at %H:%M:%S') }} 
        ({{ ((as_timestamp(now()) - as_timestamp(camera.last_changed)) / 60) | round(1) }} min) \n
        {% endfor %}
  # Additional actions: Telegram, persistent notifications, and continuous verifications
variables:
  telegram_chat_id: "<your_chat_id>"
  periodo_verificacao: "00:05:00"
  atraso_inicial: "00:01:10"
  cameras_para_monitorar:
    - camera.camera1
    - camera.camera2
    - camera.camera3
    - camera.camera4
    - camera.camera5
    - camera.camera6
    - camera.camera7
    - camera.camera8
    - camera.camera0_video_doorbell
mode: single

