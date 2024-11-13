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
    to: unavailable  # Estado que aciona a automação

# Ações a serem executadas quando o gatilho é ativado
action:
  # Ação 1: Aguardar o tempo de atraso inicial configurado
  - delay: "{{ atraso_inicial }}"

  # Ação 2: Verificar se alguma das câmeras especificadas está offline por mais de 1 minuto
  - condition: template
    value_template: >
      {% set cameras_offline = states | selectattr('entity_id', 'in', cameras_para_monitorar) | selectattr('state', 'equalto', 'unavailable') | list %}
      {% set cameras_offline_maior_1min = cameras_offline | selectattr('last_changed', 'lt', (now() - timedelta(minutes=1))) | list %}
      {{ cameras_offline_maior_1min | length > 0 }}

  # Ação 3: Enviar uma notificação no Home Assistant sobre as câmeras offline
  - service: notify.notify
    data:
      message: >
        As seguintes câmeras ficaram offline: \n
        {% set cameras_offline = states | selectattr('entity_id', 'in', cameras_para_monitorar) | selectattr('state', 'equalto', 'unavailable') | list %}
        {% for camera in cameras_offline %}
        - {{ camera.entity_id.split('.')[1].replace('_', ' ') }} - Offline desde 
        {{ (camera.last_changed | as_datetime | as_local).strftime('%d/%m/%y às %H:%M:%S') }} 
        ({{ ((as_timestamp(now()) - as_timestamp(camera.last_changed)) / 60) | round(1) }} min) \n
        {% endfor %}

  # Ação 4: Enviar uma mensagem no Telegram sobre as câmeras offline
  - service: telegram_bot.send_message
    data:
      target: "{{ telegram_chat_id }}"
      message: >
        As seguintes câmeras ficaram offline:

        {% set cameras_offline = states | selectattr('entity_id', 'in', cameras_para_monitorar) | selectattr('state', 'equalto', 'unavailable') | list %}
        {% for camera in cameras_offline %}
        - *{{ camera.entity_id.split('.')[1].replace('_', ' ') }}* - Offline desde 
        {{ (camera.last_changed | as_datetime | as_local).strftime('%d/%m/%y às %H:%M:%S') }} 
        ({{ ((as_timestamp(now()) - as_timestamp(camera.last_changed)) / 60) | round(1) }} min)
        {% endfor %}

  # Ação 5: Criar uma notificação persistente no Home Assistant sobre as câmeras offline
  - service: persistent_notification.create
    data:
      title: Câmeras Offline
      message: >
        As seguintes câmeras ficaram offline:

        {% set cameras_offline = states | selectattr('entity_id', 'in', cameras_para_monitorar) | selectattr('state', 'equalto', 'unavailable') | list %}
        {% for camera in cameras_offline %}
        - **{{ camera.entity_id.split('.')[1].replace('_', ' ') }}** - Offline desde 
        {{ (camera.last_changed | as_datetime | as_local).strftime('%d/%m/%y às %H:%M:%S') }} 
        ({{ ((as_timestamp(now()) - as_timestamp(camera.last_changed)) / 60) | round(1) }} min)
        {% endfor %}

  # Ação 6: Aguardar o período de verificação contínua (5 minutos)
  - delay: "{{ periodo_verificacao }}"

  # Ação 7: Repetir a verificação e notificações enquanto houver câmeras offline
  - repeat:
      while:
        - condition: template
          value_template: >
            {% set cameras_offline = states | selectattr('entity_id', 'in', cameras_para_monitorar) | selectattr('state', 'equalto', 'unavailable') | list %}
            {% set cameras_offline_maior_1min = cameras_offline | selectattr('last_changed', 'lt', (now() - timedelta(minutes=1))) | list %}
            {{ cameras_offline_maior_1min | length > 0 }}
      sequence:
        # Ação 7.1: Enviar notificação no Home Assistant sobre câmeras ainda offline
        - service: notify.notify
          data:
            message: >
              As seguintes câmeras ainda estão offline: \n
              {% set cameras_offline = states | selectattr('entity_id', 'in', cameras_para_monitorar) | selectattr('state', 'equalto', 'unavailable') | list %}
              {% set cameras_offline_maior_1min = cameras_offline | selectattr('last_changed', 'lt', (now() - timedelta(minutes=1))) | list %}
              {% for camera in cameras_offline_maior_1min %}
              - {{ camera.entity_id.split('.')[1].replace('_', ' ') }} - Offline desde 
              {{ (camera.last_changed | as_datetime | as_local).strftime('%d/%m/%y às %H:%M:%S') }} 
              ({{ ((as_timestamp(now()) - as_timestamp(camera.last_changed)) / 60) | round(1) }} min) \n
              {% endfor %}

        # Ação 7.2: Enviar mensagem no Telegram sobre câmeras ainda offline
        - service: telegram_bot.send_message
          data:
            target: "{{ telegram_chat_id }}"
            message: >
              As seguintes câmeras ainda estão offline:

              {% set cameras_offline = states | selectattr('entity_id', 'in', cameras_para_monitorar) | selectattr('state', 'equalto', 'unavailable') | list %}
              {% set cameras_offline_maior_1min = cameras_offline | selectattr('last_changed', 'lt', (now() - timedelta(minutes=1))) | list %}
              {% for camera in cameras_offline_maior_1min %}
              - *{{ camera.entity_id.split('.')[1].replace('_', ' ') }}* - Offline desde 
              {{ (camera.last_changed | as_datetime | as_local).strftime('%d/%m/%y às %H:%M:%S') }} 
              ({{ ((as_timestamp(now()) - as_timestamp(camera.last_changed)) / 60) | round(1) }} min)
              {% endfor %}

        # Ação 7.3: Criar uma notificação persistente no Home Assistant sobre câmeras ainda offline
        - service: persistent_notification.create
          data:
            title: Câmeras Offline (Verificação Contínua)
            message: >
              As seguintes câmeras ainda estão offline:

              {% set cameras_offline = states | selectattr('entity_id', 'in', cameras_para_monitorar) | selectattr('state', 'equalto', 'unavailable') | list %}
              {% set cameras_offline_maior_1min = cameras_offline | selectattr('last_changed', 'lt', (now() - timedelta(minutes=1))) | list %}
              {% for camera in cameras_offline_maior_1min %}
              - **{{ camera.entity_id.split('.')[1].replace('_', ' ') }}** - Offline desde 
              {{ (camera.last_changed | as_datetime | as_local).strftime('%d/%m/%y às %H:%M:%S') }} 
              ({{ ((as_timestamp(now()) - as_timestamp(camera.last_changed)) / 60) | round(1) }} min)
              {% endfor %}

        # Ação 7.4: Aguardar o período de verificação contínua novamente
        - delay: "{{ periodo_verificacao }}"

  # Ação 8: Notificar que todas as câmeras voltaram ao estado online após a restauração
  - service: notify.notify
    data:
      message: Todas as câmeras estão online.

  # Ação 9: Enviar mensagem no Telegram indicando que todas as câmeras estão online
  - service: telegram_bot.send_message
    data:
      target: "{{ telegram_chat_id }}"
      message: Todas as câmeras estão *online*.

  # Ação 10: Criar uma notificação persistente indicando que todas as câmeras estão online
  - service: persistent_notification.create
    data:
      title: Estado Operacional Online
      message: Todas as câmeras estão **online**.

# Definindo variáveis usadas nas ações
variables:
  telegram_chat_id: "**************"  # Substitua pelo seu chat_id
  periodo_verificacao: "00:05:00"  # Período de verificação contínua (5 minutos)
  atraso_inicial: "00:01:10"  # Atraso inicial para aguardar 1 minuto antes da primeira verificação
  cameras_para_monitorar:
    - camera.camera1
    - camera.camera2
    - camera.camera3
    - camera.camera4
    - camera.camera5
    - camera.camera6
    - camera.camera7
    - camera.camera8
    - camera.camera0_video_doorbell  # Lista de câmeras para monitorar

# Configurando o modo de execução da automação
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
# Automation Name and Description
alias: Notify and Check Offline Cameras
description: >
  Notifies on Telegram, the Home Assistant app, and Home Assistant interface
  when cameras go offline, with periodic notifications while the cameras
  remain offline.

# Triggers to activate the automation when any camera in the list goes offline
triggers:
  - entity_id:
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
    trigger: state

# Actions executed when the trigger is activated
actions:
  - delay: "{{ initial_delay }}" # Waits for the configured initial delay
  - condition: template
    value_template: >
      {% set offline_cameras = states | selectattr('entity_id', 'in',
      cameras_to_monitor) | selectattr('state', 'equalto', 'unavailable') |
      list %} {% set offline_cameras_more_than_1min = offline_cameras |
      selectattr('last_changed', 'lt', (now() - timedelta(minutes=1))) | list %}
      {{ offline_cameras_more_than_1min | length > 0 }}

  # Notification to the Home Assistant app listing offline cameras
  - data:
      message: >
        The following cameras went offline: \n {% set offline_cameras = states
        | selectattr('entity_id', 'in', cameras_to_monitor) |
        selectattr('state', 'equalto', 'unavailable') | list %} {% for camera in
        offline_cameras %} - {{ camera.entity_id.split('.')[1].replace('_', ' ')
        }} - Offline since {{ (camera.last_changed | as_datetime |
        as_local).strftime('%d/%m/%y at %H:%M:%S') }} ({{ ((as_timestamp(now())
        - as_timestamp(camera.last_changed)) / 60) | round(1) }} min) \n {%
        endfor %}
    action: notify.notify

  # Telegram notification with offline camera details
  - data:
      target: "{{ telegram_chat_id }}"
      message: >
        The following cameras went offline:

        {% set offline_cameras = states | selectattr('entity_id', 'in',
        cameras_to_monitor) | selectattr('state', 'equalto', 'unavailable')
        | list %} {% for camera in offline_cameras %} - *{{
        camera.entity_id.split('.')[1].replace('_', ' ') }}* - Offline since {{
        (camera.last_changed | as_datetime | as_local).strftime('%d/%m/%y at
        %H:%M:%S') }} ({{ ((as_timestamp(now()) -
        as_timestamp(camera.last_changed)) / 60) | round(1) }} min)

        {% endfor %}
    action: telegram_bot.send_message

  # Persistent notification on Home Assistant interface
  - data:
      title: Offline Cameras
      message: >
        The following cameras went offline:

        {% set offline_cameras = states | selectattr('entity_id', 'in',
        cameras_to_monitor) | selectattr('state', 'equalto', 'unavailable')
        | list %} {% for camera in offline_cameras %} - **{{
        camera.entity_id.split('.')[1].replace('_', ' ') }}** - Offline since
        {{ (camera.last_changed | as_datetime | as_local).strftime('%d/%m/%y at
        %H:%M:%S') }} ({{ ((as_timestamp(now()) -
        as_timestamp(camera.last_changed)) / 60) | round(1) }} min)

        {% endfor %}
    action: persistent_notification.create

  # Waits for the verification period
  - delay: "{{ verification_period }}"

  # Repeats notifications every 5 minutes as long as there are offline cameras
  - repeat:
      while:
        - condition: template
          value_template: >
            {% set offline_cameras = states | selectattr('entity_id', 'in',
            cameras_to_monitor) | selectattr('state', 'equalto',
            'unavailable') | list %} {% set offline_cameras_more_than_1min =
            offline_cameras | selectattr('last_changed', 'lt', (now() -
            timedelta(minutes=1))) | list %} {{ offline_cameras_more_than_1min |
            length > 0 }}
      sequence:
        - data:
            message: >
              The following cameras are still offline: \n {% set
              offline_cameras = states | selectattr('entity_id', 'in',
              cameras_to_monitor) | selectattr('state', 'equalto',
              'unavailable') | list %} {% set offline_cameras_more_than_1min =
              offline_cameras | selectattr('last_changed', 'lt', (now() -
              timedelta(minutes=1))) | list %} {% for camera in
              offline_cameras_more_than_1min %} - {{
              camera.entity_id.split('.')[1].replace('_', ' ') }} - Offline since
              {{ (camera.last_changed | as_datetime |
              as_local).strftime('%d/%m/%y at %H:%M:%S') }} ({{ ((as_timestamp(now()) -
              as_timestamp(camera.last_changed)) / 60) | round(1) }} min) \n
              {% endfor %}
          action: notify.notify

        - data:
            target: "{{ telegram_chat_id }}"
            message: >
              The following cameras are still offline:

              {% set offline_cameras = states | selectattr('entity_id', 'in',
              cameras_to_monitor) | selectattr('state', 'equalto',
              'unavailable') | list %} {% set offline_cameras_more_than_1min =
              offline_cameras | selectattr('last_changed', 'lt', (now() -
              timedelta(minutes=1))) | list %} {% for camera in
              offline_cameras_more_than_1min %} - *{{
              camera.entity_id.split('.')[1].replace('_', ' ') }}* - Offline
              since {{ (camera.last_changed | as_datetime |
              as_local).strftime('%d/%m/%y at %H:%M:%S') }} ({{ ((as_timestamp(now()) -
              as_timestamp(camera.last_changed)) / 60) | round(1) }} min)

              {% endfor %}
          action: telegram_bot.send_message

        - data:
            title: Offline Cameras (Continuous Verification)
            message: >
              The following cameras are still offline:

              {% set offline_cameras = states | selectattr('entity_id', 'in',
              cameras_to_monitor) | selectattr('state', 'equalto',
              'unavailable') | list %} {% set offline_cameras_more_than_1min =
              offline_cameras | selectattr('last_changed', 'lt', (now() -
              timedelta(minutes=1))) | list %} {% for camera in
              offline_cameras_more_than_1min %} - **{{
              camera.entity_id.split('.')[1].replace('_', ' ') }}** - Offline
              since {{ (camera.last_changed | as_datetime |
              as_local).strftime('%d/%m/%y at %H:%M:%S') }} ({{ ((as_timestamp(now()) -
              as_timestamp(camera.last_changed)) / 60) | round(1) }} min)

              {% endfor %}
          action: persistent_notification.create

        - delay: "{{ verification_period }}"

  # Notify that all cameras are online again
  - data:
      message: All cameras are online.
    action: notify.notify

  - data:
      target: "{{ telegram_chat_id }}"
      message: All cameras are *online*.
    action: telegram_bot.send_message

  - data:
      title: Operational State Online
      message: All cameras are **online**.
    action: persistent_notification.create

# Variables
variables:
  telegram_chat_id: "-***************"
  verification_period: "00:05:00"
  initial_delay: "00:01:10"
  cameras_to_monitor:
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
