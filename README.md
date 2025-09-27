# Esp32-POE-multi-capteurs

Je vous présente ici la version POE de l'esp32 multi capteurs avec bluetooth proxy

<img width="801" height="514" alt="image" src="https://github.com/user-attachments/assets/780eb400-ffa3-4aaa-91db-b5f4fc6185d0" />


Matériel utilisé:
 - Olimex ESP32-POE-ISO
 - DHT22 ( capteur de température / humidite )
 - BH1750 ( capteur de luminosité )
 - LD2410 ( radar millimétrique )

Généralités

Ce code est prévu pour ESPHome, une solution qui permet de créer un firmware personnalisé pour les microcontrôleurs compatibles avec Home Assistant.
Le module utilisé est l’Olimex ESP32-POE-ISO, qui a la particularité de se connecter via Ethernet et d'intégrer des fonctionnalités avancées pour la domotique et la détection de présence Bluetooth.

Substitutions & Généralités
    substitutions: Définit des variables réutilisables (name, friendly_name) pour faciliter la maintenance et la personnalisation.
    esp32 : Spécifie la carte (esp32dev) et le framework (esp-idf) utilisé.
    esphome : Nom du projet, suffixe du nom désactivé, nom convivial.
    ota : Active les mises à jour "Over The Air" via ESPHome.
    api : Active l’API sécurisée vers Home Assistant, avec clé d’encryption.
    logger : Affiche les logs de niveau INFO pour le débogage.

Connexion Réseau Ethernet
    ethernet : Configure LAN8720 pour la connexion filaire, les pins utilisés pour MDC, MDIO et CLK, et l'adresse PHY.
    Offre aussi la possibilité de fixer une IP statique (manual_ip) avec gateway et subnet, à adapter selon le réseau utilisé.

Bus I2C
    i2c : Active le bus I2C sur les broches SDA 13, SCL 16 et permet le scan automatique de certains devices.

Capteurs textuels (Text Sensor)
    Affiche l’IP et l’adresse MAC du module, la version du firmware, et l’uptime formaté (jours, heures, minutes) en texte.
    Le capteur d’uptime utilise une lambda pour la conversion du temps en format humain.

Switch
    switch : Permet de redémarrer le module depuis l’interface Home Assistant.

Bluetooth Proxy & Tracker
    Active le proxy Bluetooth pour capter les signaux BLE et servir de passerelle pour Home Assistant.
    esp32_ble_tracker : Paramètres du scan (intervalle 15s, actif).
    bluetooth_proxy : Active la fonctionnalité de proxy BLE.

Détection de présence Bluetooth (Binary Sensor)
    Trois entités de présence BLE sont définies (M1/M2 + leur MAC). Les paramètres timeout et min_rssi ajustent respectivement la durée avant qu’un appareil soit considéré absent et le seuil de détection (distance).
    Ajoute le statut global du module ESP.
    Capteurs du radar LD2410 : cible détectée, cible mobile, cible fixe.

UART (Interface Série)
    Configure l’UART utilisé pour communiquer avec le radar LD2410 (pins TX/RX, débit de 256000 bauds).

Radar LD2410
    Déclare l’intégration radar LD2410 pour détecter présente, mouvements et valeurs de distance/énergie.
    Crée de nombreux objets "number" pour régler les seuils de distance et d’énergie sur les différents "gates" (zones).

Sensors (Capteurs analogiques et numériques)
    Mémoire libre ESP, température interne, uptime brut, RSSI BLE pour mesurer la force du signal, tous les attributs du radar LD2410.
    Capteur de température/humidité DHT sur GPIO33, avec un offset pour corriger légèrement la température.
    Capteur de luminosité BH1750 sur I2C adresse 0x23.

À adapter avant usage
    Remplacer les adresses MAC, clés d’encryption et paramètres réseau par les valeurs réelles de votre installation.
    Adapter les noms et descriptions selon vos besoins.

Usage typique

Ce fichier YAML doit être utilisé dans ESPHome, puis intégré à Home Assistant via l’API ou un upload OTA.
Permet de réaliser une passerelle Bluetooth avancée sur Ethernet, tout en intégrant un suivi radar, des mesures environnementales et un monitoring complet du module ESP32.


Le code yaml de base utilisé
   
```yaml
substitutions:
  name: olimex-esp32-poe-iso-706000
  friendly_name: Bluetooth Proxy 706000

esp32:
  board: esp32dev
  framework:
    type: esp-idf

ota:
  - platform: esphome

esphome:
  name: ${name}
  name_add_mac_suffix: false
  friendly_name: ${friendly_name}
api:
  encryption:
    key: cleDeCryptage # à modifer

ethernet:
  type: LAN8720
  mdc_pin: GPIO23
  mdio_pin: GPIO18
  clk:
    mode: CLK_OUT
    pin: GPIO17
  phy_addr: 0
  power_pin: GPIO12
  manual_ip:
    static_ip: adresseIp # à modifer
    gateway: passerelle # à modifer
    subnet: sousReseau # à modifer

i2c:
  sda: 13
  scl: 16
  scan: true
  id: bus_a

text_sensor:
  - platform: ethernet_info
    ip_address:
      name: "IP Address"
    mac_address:
      name: "MAC Address"
  - platform: version
    name: "${name}_version"
  - platform: template
    name: "Uptime (Jours, Heures et Minutes)"
    lambda: |-
      int seconds = id(uptime_sensor).state;
      int days = seconds / 86400;
      seconds = seconds % 86400;
      int hours = seconds / 3600;
      seconds = seconds % 3600;
      int minutes = seconds / 60;

      // Renvoie la chaîne formatée
      return (days > 0 ? std::to_string(days) + " jours " : "") + 
             (hours > 0 ? std::to_string(hours) + " h " : "") +
             (minutes > 0 ? std::to_string(minutes) + " min" : "0 min");
    update_interval: 60s
    entity_category: "diagnostic"  

switch:
  - platform: restart
    name: "Restart"

# Enable logging
logger:
  level: INFO

esp32_ble_tracker:
  scan_parameters:
    duration: 30s
    active: true

bluetooth_proxy:
  active: true

binary_sensor:
  - platform: ble_presence
    mac_address: adresseMAC1 # à modifer
    name: "Mi_band_1"
    timeout: 30s # le temps avant que ca passe en absent
    min_rssi: -70dB # permet de regler la "distance" de detection
    id: Mi_band_1  
  - platform: ble_presence
    mac_address: adresseMAC2 # à modifer
    name: "Mi_band_2"
    timeout: 30s # le temps avant que ca passe en absent
    min_rssi: -70dB # permet de regler la "distance" de detection
    id: Mi_band_2
  - platform: status
    name: "Status"
  - platform: ld2410
    has_target:
      name: "Radar Target"
      id: radar_has_target
    has_moving_target:
      name: "Radar Moving Target"
      id: radar_moving_target
    has_still_target:
      name: "Radar Still Target"
      id: radar_still_target

uart:
  id: ld2410_uart
  tx_pin: 32
  rx_pin: 36
  baud_rate: 256000
  parity: NONE
  stop_bits: 1

ld2410:
  uart_id: ld2410_uart
#  throttle: 1500ms
  id: ld2410_comp

number:
  - platform: ld2410
    timeout:
      name: Radar Timeout
    max_move_distance_gate:
      name: Radar Max Move Distance
    max_still_distance_gate:
      name: Radar Max Still Distance
    g0:
      move_threshold:
        name: g0 move threshold
      still_threshold:
        name: g0 still threshold
    g1:
      move_threshold:
        name: g1 move threshold
      still_threshold:
        name: g1 still threshold
    g2:
      move_threshold:
        name: g2 move threshold
      still_threshold:
        name: g2 still threshold
    g3:
      move_threshold:
        name: g3 move threshold
      still_threshold:
        name: g3 still threshold
    g4:
      move_threshold:
        name: g4 move threshold
      still_threshold:
        name: g4 still threshold
    g5:
      move_threshold:
        name: g5 move threshold
      still_threshold:
        name: g5 still threshold
    g6:
      move_threshold:
        name: g6 move threshold
      still_threshold:
        name: g6 still threshold
    g7:
      move_threshold:
        name: g7 move threshold
      still_threshold:
        name: g7 still threshold
    g8:
      move_threshold:
        name: g8 move threshold
      still_threshold:
        name: g8 still threshold

sensor:
  - platform: template
    id: esp_memory
    icon: mdi:memory
    name: ESP Free Memory
    lambda: return heap_caps_get_free_size(MALLOC_CAP_INTERNAL) / 1024;
    unit_of_measurement: 'kB'
    state_class: measurement
    entity_category: "diagnostic"
  - platform: internal_temperature
    name: "intern_temp"
  - platform: uptime
    name: "Uptime Raw"
    id: uptime_sensor
  - platform: ble_rssi
    mac_address: adresseMAC1 # à modifer
    name: "Mi_band_laurent Min RSSI"

  - platform: ld2410
    moving_distance:
      name: Radar Moving Distance
      id: moving_distance
    still_distance:
      name: Radar Still Distance
      id: still_distance
    moving_energy:
      name: Radar Move Energy
    still_energy:
      name: Radar Still Energy
    detection_distance:
      name: Radar Detection Distance
      id: radar_detection_distance
    g0:
      move_energy:
        name: g0 move energy
      still_energy:
        name: g0 still energy
    g1:
      move_energy:
        name: g1 move energy
      still_energy:
        name: g1 still energy
    g2:
      move_energy:
        name: g2 move energy
      still_energy:
        name: g2 still energy
    g3:
      move_energy:
        name: g3 move energy
      still_energy:
        name: g3 still energy
    g4:
      move_energy:
        name: g4 move energy
      still_energy:
        name: g4 still energy
    g5:
      move_energy:
        name: g5 move energy
      still_energy:
        name: g5 still energy
    g6:
      move_energy:
        name: g6 move energy
      still_energy:
        name: g6 still energy
    g7:
      move_energy:
        name: g7 move energy
      still_energy:
        name: g7 still energy
    g8:
      move_energy:
        name: g8 move energy
      still_energy:
        name: g8 still energy

  - platform: dht
    pin: 33
    temperature:
      name: "Temperature1"
      accuracy_decimals: 2
      device_class: "temperature"
      filters:
        - offset: offset 0.0 # à modifer si besoin

    humidity:
      name: "Humidite1"
      accuracy_decimals: 2
      device_class: "humidity"
    update_interval: 20s
    
  - platform: bh1750
    name: "Lumiere1"
    device_class: "illuminance"
    address: 0x23
    update_interval: 60s
