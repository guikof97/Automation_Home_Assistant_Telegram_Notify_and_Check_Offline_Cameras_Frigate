# Automação Home Assistant: Notificação de Câmeras Offline Home Assistant e Frigate

Este projeto é uma automação para o [Home Assistant](https://www.home-assistant.io/), que monitora o estado de câmeras configuradas no sistema. Ele envia notificações via Telegram, no aplicativo do Home Assistant e na interface de notificações persistentes quando uma ou mais câmeras ficam offline. A automação também verifica periodicamente se as câmeras permanecem offline e notifica novamente, até que todas as câmeras retornem ao estado online.

---

## Funcionalidades

1. **Detecção de câmeras offline:** A automação monitora o estado de câmeras e identifica quando mudam para `unavailable`, `offline` ou `disconnected`.
2. **Notificação inicial:** Quando uma ou mais câmeras permanecem offline por mais de 1 minuto, uma notificação é enviada.
3. **Notificações periódicas:** Enquanto as câmeras estiverem offline, a automação envia notificações a cada 5 minutos.
4. **Notificação de retorno ao estado online:** Após todas as câmeras voltarem ao estado online, é enviada uma notificação informando o retorno ao estado operacional.

---

## Variáveis

- **`telegram_chat_id`**: ID do chat do Telegram para onde as notificações serão enviadas.
- **`periodo_verificacao`**: Intervalo de tempo entre as verificações de câmeras offline (padrão: 5 minutos).
- **`atraso_inicial`**: Tempo de espera inicial para verificar se uma câmera permanece offline (padrão: 1 minuto).

---

## Pré-requisitos

- Instalação do [Home Assistant](https://www.home-assistant.io/).
- Integração configurada do **Telegram Bot** no Home Assistant.
- Câmeras devidamente configuradas como entidades no Home Assistant.

---

## Configuração

### Substitua no código:

1. `telegram_chat_id`: Adicione o ID do seu chat do Telegram.
2. `entity_id` das câmeras no campo `triggers`: Inclua as entidades das câmeras que você deseja monitorar.

---

## Exemplo de Automação

Aqui está o código completo da automação:

```yaml
alias: Notificar e Verificar Câmeras Offline
description: >
  Notifica no Telegram, no aplicativo e na interface do Home Assistant quando
  câmeras ficam offline, com notificações periódicas enquanto as câmeras
  estiverem offline.
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
    platform: state
actions:
  - delay: "{{ atraso_inicial }}"
  - condition: template
    value_template: >
      {% set cameras_offline = states.camera | selectattr('state', 'in', ['unavailable', 'offline', 'disconnected']) | list %} 
      {% set cameras_offline_mais_1min = cameras_offline | selectattr('last_changed', 'lt', (now() - timedelta(minutes=1))) | list %} 
      {{ cameras_offline_mais_1min | length > 0 }}
  - variables:
      mensagem_notificacao: >-
        As seguintes câmeras estão offline:
        {% set cameras_offline = states.camera | selectattr('state', 'in', ['unavailable', 'offline', 'disconnected']) | list %} {%
        set cameras_offline_mais_1min = cameras_offline | selectattr('last_changed', 'lt', (now() - timedelta(minutes=1))) | list %} {%
        for camera in cameras_offline_mais_1min %}
          - *{{ camera.entity_id.split('.')[1].replace('_', ' ') }}*: {{
           (camera.last_changed | as_datetime | as_local).strftime('%d/%m/%y às %H:%M:%S') }} ({{
            ((as_timestamp(now()) - as_timestamp(camera.last_changed)) / 60) | round(1) }} min). {% endfor %}
  - service: telegram_bot.send_message
    data:
      target: "{{ telegram_chat_id }}"
      message: "{{ mensagem_notificacao }}"
  - service: persistent_notification.create
    data:
      title: Câmeras Offline
      message: "{{ mensagem_notificacao | replace('*', '**') }}"
  - service: notify.notify
    data:
      message: |-
        {{ mensagem_notificacao | replace('*', '') }}
  - delay: "{{ periodo_verificacao }}"
  - repeat:
      while:
        - condition: template
          value_template: >
            {% set cameras_offline = states.camera | selectattr('state', 'in', ['unavailable', 'offline', 'disconnected']) | list %} 
            {% set cameras_offline_mais_1min = cameras_offline | selectattr('last_changed', 'lt', (now() - timedelta(minutes=1))) | list %} 
            {{ cameras_offline_mais_1min | length > 0 }}
      sequence:
        - variables:
            mensagem_notificacao2: >-
              As seguintes câmeras ainda estão offline:
              {% set cameras_offline = states.camera | selectattr('state', 'in', ['unavailable', 'offline', 'disconnected']) | list %} {%
              set cameras_offline_mais_1min = cameras_offline | selectattr('last_changed', 'lt', (now() - timedelta(minutes=1))) | list %} {%
              for camera in cameras_offline_mais_1min %}
                - *{{ camera.entity_id.split('.')[1].replace('_', ' ') }}*: {{
                 (camera.last_changed | as_datetime | as_local).strftime('%d/%m/%y às %H:%M:%S') }} ({{
                  ((as_timestamp(now()) - as_timestamp(camera.last_changed)) / 60) | round(1) }} min). {% endfor %}
        - service: telegram_bot.send_message
          data:
            target: "{{ telegram_chat_id }}"
            message: "{{ mensagem_notificacao2 }}"
        - service: persistent_notification.create
          data:
            title: Câmeras Offline (Verificação Contínua)
            message: "{{ mensagem_notificacao2 | replace('*', '**') }}"
        - service: notify.notify
          data:
            message: |-
              {{ mensagem_notificacao2 | replace('*', '') }}
        - delay: "{{ periodo_verificacao }}"
  - service: notify.notify
    data:
      message: Todas as câmeras estão online.
  - service: telegram_bot.send_message
    data:
      target: "{{ telegram_chat_id }}"
      message: Todas as câmeras estão *online*.
  - service: persistent_notification.create
    data:
      title: Estado Operacional Online
      message: Todas as câmeras estão **online**.
variables:
  telegram_chat_id: "SEU_CHAT_ID"
  periodo_verificacao: "00:05:00"
  atraso_inicial: "00:01:05"
mode: single
```
---

## Funcionamento do Código

1. **Gatilhos (`triggers`)**: 
   - Monitora câmeras específicas e dispara a automação quando alguma delas muda para o estado `unavailable`.
   
2. **Ação de atraso inicial (`delay`)**: 
   - Garante que o estado offline seja mantido por pelo menos 1 minuto antes de processar notificações.

3. **Verificação de estado offline (`condition`)**: 
   - Confirma que uma ou mais câmeras estão offline há mais de 1 minuto.

4. **Notificações**:
   - **Telegram**: Envia uma mensagem detalhada sobre as câmeras offline.
   - **Notificação persistente**: Aparece na interface do Home Assistant como notificação permanente.
   - **Notificação no app**: Enviada como alerta no aplicativo do Home Assistant.

5. **Repetição periódica (`repeat`)**: 
   - Notifica novamente enquanto as câmeras permanecerem offline.

6. **Notificação de retorno (`notify`)**: 
   - Informa quando todas as câmeras voltam ao estado online.

---

## Licença

Este projeto é fornecido sob a licença MIT. Consulte o arquivo [LICENSE](LICENSE) para mais detalhes.



---

# Home Assistant Automation: Offline Camera Notification Home Assistant and Frigate

This project is an automation for [Home Assistant](https://www.home-assistant.io/), which monitors the state of cameras configured in the system. It sends notifications via Telegram, the Home Assistant app, and the persistent notification interface when one or more cameras go offline. The automation also periodically checks if the cameras remain offline and sends notifications again until all cameras return to the online state.

---

## Features

1. **Offline Camera Detection:** The automation monitors the state of cameras and identifies when they change to `unavailable`, `offline`, or `disconnected`.
2. **Initial Notification:** When one or more cameras have been offline for over 1 minute, a notification is sent.
3. **Periodic Notifications:** As long as the cameras are offline, the automation sends notifications every 5 minutes.
4. **Return to Online State Notification:** Once all cameras return to the online state, a notification is sent indicating the return to operational status.

---

## Variables

- **`telegram_chat_id`**: The Telegram chat ID where the notifications will be sent.
- **`verification_interval`**: Time interval between checks for offline cameras (default: 5 minutes).
- **`initial_delay`**: The initial waiting time before checking if a camera remains offline (default: 1 minute).

---

## Prerequisites

- [Home Assistant](https://www.home-assistant.io/) installed.
- Telegram Bot integration configured in Home Assistant.
- Cameras properly configured as entities in Home Assistant.

---

## Configuration

### Replace the following in the code:

1. `telegram_chat_id`: Add your own Telegram chat ID.
2. Camera `entity_id` in the `triggers` section: Include the entities of the cameras you want to monitor.

---

## Example Automation

Here is the full automation code:

```yaml
alias: Notify and Check Offline Cameras
description: >
  Sends notifications on Telegram, the app, and the Home Assistant interface
  when cameras go offline, with periodic notifications while cameras are offline.
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
    platform: state
actions:
  - delay: "{{ initial_delay }}"
  - condition: template
    value_template: >
      {% set offline_cameras = states.camera | selectattr('state', 'in', ['unavailable', 'offline', 'disconnected']) | list %} 
      {% set offline_cameras_older_than_1min = offline_cameras | selectattr('last_changed', 'lt', (now() - timedelta(minutes=1))) | list %} 
      {{ offline_cameras_older_than_1min | length > 0 }}
  - variables:
      notification_message: >-
        The following cameras are offline:
        {% set offline_cameras = states.camera | selectattr('state', 'in', ['unavailable', 'offline', 'disconnected']) | list %} {%
        set offline_cameras_older_than_1min = offline_cameras | selectattr('last_changed', 'lt', (now() - timedelta(minutes=1))) | list %} {%
        for camera in offline_cameras_older_than_1min %}
          - *{{ camera.entity_id.split('.')[1].replace('_', ' ') }}*: {{
           (camera.last_changed | as_datetime | as_local).strftime('%d/%m/%y at %H:%M:%S') }} ({{
            ((as_timestamp(now()) - as_timestamp(camera.last_changed)) / 60) | round(1) }} min). {% endfor %}
  - service: telegram_bot.send_message
    data:
      target: "{{ telegram_chat_id }}"
      message: "{{ notification_message }}"
  - service: persistent_notification.create
    data:
      title: Offline Cameras
      message: "{{ notification_message | replace('*', '**') }}"
  - service: notify.notify
    data:
      message: |-
        {{ notification_message | replace('*', '') }}
  - delay: "{{ verification_interval }}"
  - repeat:
      while:
        - condition: template
          value_template: >
            {% set offline_cameras = states.camera | selectattr('state', 'in', ['unavailable', 'offline', 'disconnected']) | list %} 
            {% set offline_cameras_older_than_1min = offline_cameras | selectattr('last_changed', 'lt', (now() - timedelta(minutes=1))) | list %} 
            {{ offline_cameras_older_than_1min | length > 0 }}
      sequence:
        - variables:
            notification_message2: >-
              The following cameras are still offline:
              {% set offline_cameras = states.camera | selectattr('state', 'in', ['unavailable', 'offline', 'disconnected']) | list %} {%
              set offline_cameras_older_than_1min = offline_cameras | selectattr('last_changed', 'lt', (now() - timedelta(minutes=1))) | list %} {%
              for camera in offline_cameras_older_than_1min %}
                - *{{ camera.entity_id.split('.')[1].replace('_', ' ') }}*: {{
                 (camera.last_changed | as_datetime | as_local).strftime('%d/%m/%y at %H:%M:%S') }} ({{
                  ((as_timestamp(now()) - as_timestamp(camera.last_changed)) / 60) | round(1) }} min). {% endfor %}
        - service: telegram_bot.send_message
          data:
            target: "{{ telegram_chat_id }}"
            message: "{{ notification_message2 }}"
        - service: persistent_notification.create
          data:
            title: Offline Cameras (Continuous Check)
            message: "{{ notification_message2 | replace('*', '**') }}"
        - service: notify.notify
          data:
            message: |-
              {{ notification_message2 | replace('*', '') }}
        - delay: "{{ verification_interval }}"
  - service: notify.notify
    data:
      message: All cameras are online.
  - service: telegram_bot.send_message
    data:
      target: "{{ telegram_chat_id }}"
      message: All cameras are *online*.
  - service: persistent_notification.create
    data:
      title: Operational State Online
      message: All cameras are **online**.
variables:
  telegram_chat_id: "YOUR_CHAT_ID"
  verification_interval: "00:05:00"
  initial_delay: "00:01:05"
mode: single
```
---

## Code Functionality

1. **Triggers (`triggers`)**: 
   - Monitors specific cameras and triggers the automation when any of them change to the `unavailable` state.
   
2. **Initial Delay (`delay`)**: 
   - Ensures that the offline state is maintained for at least 1 minute before processing notifications.

3. **Offline State Check (`condition`)**: 
   - Verifies that one or more cameras have been offline for more than 1 minute.

4. **Notifications**:
   - **Telegram**: Sends a detailed message about the offline cameras.
   - **Persistent Notification**: Appears in the Home Assistant interface as a permanent notification.
   - **App Notification**: Sends an alert to the Home Assistant app.

5. **Periodic Repetition (`repeat`)**: 
   - Sends additional notifications as long as the cameras are offline.

6. **Return to Online Notification (`notify`)**: 
   - Informs when all cameras have returned to the online state.

---

## License

This project is provided under the MIT license. See the [LICENSE](LICENSE) file for more details.

