# Ilumin8
Um projeto academico construido com a placa ESP32-rev1 com o objetivo de ajudar na economia de energia através do controle dinâmico de lâmpadas.

O projeto funciona basicamente utilizando um sensor de presença para detectar o movimento, e então envia as informações para o Broker MQTT and liga o LED (Atuador utilizado neste projeto para motivos demonstrativos) representando a ligação das luzes do sistema.

# Como reproduzir

O código aqui armazenado e documentado foi testado apenas e unicamente com os dispositivos descritos na seção de [Hardware](#Hardware), qualquer mudança no modelo destes dispositivos pode ou não exigir alterações no código fonte. Além disso, as instruções aqui contidas ja levam em conta que você ja tem os pacotes do Arduino IDE necessários instalados para inserir código na placa em questão. Com adendos colocados, para reproduzir o projeto, siga os seguintes passos:

### Montagem
1 - Conecte o sensor de presença na placa de acordo com suas terminações especificas, com uma das conexões para o sinal.
2 - Conecte o led na placa de acordo com suas terminações especificas, é recomendavel utilizar um resistor entre os dois para evitar uma possivel queima do mesmo.

### Codificação
1. Clone este repositorio
2. Abra o arquivo "ilumin8.ino" contido na pasta "/src" com o Arduino IDE
3. Subsititua as informações das variaveis SSID e PASSWORD, com as credenciais de acesso da rede Wireless a ser utilizada, respectivamente: Nome da rede e Senha.
4. Substitua os valores numericos de LAMP_PIN e PIR_PIN com a identificação das portas escolhidas para conectar, respectivamente o LED e o Sensor de presença.
6. Caso desejado, substitua os valores de BROKER_MQTT e BROKER_PORT com, respectivamente o endereço e porta do Broker MQTT desejado, caso contrário sinta-se livre para deixar os valores padrão, uma vez que estamos utilizando um servidor de teste Mosquitto gratuito.
5. Baixe o código para a placa em questão

### Broker MQTT
1. Baixe o App MQTT Dashboard para Android (Recomendado, mas sinta-se livre para utilizar o aplicativo de sua preferência)
2. Configure o dashboard com a porta e o endereço utilizado.

# Software

The core code has been developed in C++ with the Arduino IDE

```c++
// Importa bibliotecas de WiFi e suporte ao MQTT
# include <WiFi.h>
# include <PubSubClient.h>

// Define pinos dos dispositivos utilizados
# define LAMP_PIN 33
# define PIR_PIN 32

// Define identificador e topicos utilizados pelo MQTT
# define ID_MQTT "ilumin8_mqtt"
# define TOPICO_SUBSCRIBE_LAMPADA "topico_liga_desliga_lampada"
# define TOPICO_PUBLISH_MONITORAMENTO "topico_sensor_pir"

// Credenciais para entrar na rede WiFi e se conectar à internet, substitua por suas credenciais
const char * SSID = "REPLACE";
const char * PASSWORD = "REPLACE";

// Informações do broker MQTT a ser utilizado
const char * BROKER_MQTT = "test.mosquitto.org";
int BROKER_PORT = 1883;

// Variaveis de controle de estado
int last_change = 0;
int pir_state = 0;
int led_state = 0;
boolean beacon = false;
char * payload = "";

// Instanciando objetos da conexão WiFi e MQTT
WiFiClient espClient;
PubSubClient MQTT(espClient);

// Instanciando funções necessárias
void IniciaMQTT(void);
void MQTTCallback(char * topic, byte * payload, unsigned int length);
void ReconectaMQTT(void);
void ReconectaWiFi(void);
void VerificarConexoes(void);

// Função que inicia os parametros do MQTT
void IniciaMQTT(void) {
  MQTT.setServer(BROKER_MQTT, BROKER_PORT);
  MQTT.setCallback(MQTTCallback);
}

// Função que é chamada sempre que uma mensagem é recebida pelo dispositivo
void MQTTCallback(char * topic, byte * payload, unsigned int length) {
  String msg;

  // Obtem a mensagem do payload recebido
  for (int i = 0; i < length; i++) {
    char c = (char) payload[i];
    msg += c;
  }

  // Imprime a mensagem recebida
  Serial.print("Chegou a seguinte string via MQTT: ");
  Serial.println(msg);

  // Valida qual ação deve ser tomada de acordo com a mensagem recebida
  // Se for 1, liga o/a led/lâmpada
  if (msg.equals("1")) {
    digitalWrite(LAMP_PIN, HIGH);
    Serial.print("LED aceso mediante comando MQTT");
  }
  // Se for 0, desliga o/a led/lâmpada
  if (msg.equals("0")) {
    digitalWrite(LAMP_PIN, LOW);
    Serial.print("LED apagado mediante comando MQTT");
  }
}

// Função que conecta/reconecta o dispositivo ao broker MQTT
void ReconectaMQTT(void) {
  // Se não há conexão com o Broker, a conexão é refeita
  if (!MQTT.connected()){
    // Enquanto o MQTT nao estiver conectado
    while (!MQTT.connected()) {
      Serial.print("* Tentando se conectar ao Broker MQTT: ");
      Serial.println(BROKER_MQTT);
      // Caso esteja conectado ao ID especificado em ID_MQTT
      if (MQTT.connect(ID_MQTT)) {
        // Se inscreve ao topico da lâmpada
        Serial.println("Conectado com sucesso ao broker MQTT!");
        MQTT.subscribe(TOPICO_SUBSCRIBE_LAMPADA);
      } else {
        // Caso contrário, aguarda 2 segundos e tenta novamente
        Serial1.println("Falha ao reconectar no broker.");
        Serial.println("Havera nova tentatica de conexao em 2s");
        delay(2000);
      }
    }
  }
}

// Função que reconecta o serviço de WiFi
void ReconectaWiFi(void) {

  // Se há conexão com o WiFi, a função retorna
  if (WiFi.status() == WL_CONNECTED)
    return;

  // Inicia o serviço WiFi com as credenciais providas
  WiFi.begin(SSID, PASSWORD);

  // Verifica se a conexão foi finalizada em um intervalo de 100ms
  while (WiFi.status() != WL_CONNECTED) {
    delay(100);
    Serial.print(".");
  }

  Serial.println();
  Serial.print("Conectado com sucesso na rede ");
  Serial.print(SSID);
  Serial.println("IP obtido: ");
  Serial.println(WiFi.localIP());
}

// Função que é rodada em cada iteração para reconectar os serviços que estejam desconectados
void VerificaConexoes(void) {
  ReconectaWiFi(); 
  ReconectaMQTT(); 
}

// Função responsavel por ler os dados do sensor PIR e alterar dados de acordo com a leitura
void ChecaMovimento(void) {

  // Le o sensor PIR no pino especificado
  pir_state = digitalRead(PIR_PIN);

  // Caso a leitura mais recente seja diferente da ultima leitura (Interrupção)
  if (pir_state != last_change) {
    // Caso a leitura seja 1/HIGH o payload 1 é escrito
    if (pir_state == HIGH) {
      led_state = HIGH;
      payload = "1";
    } 
    // Caso a leitura seja 0/LOW o payload 0 é escrito
    else {
      led_state = LOW;
      payload = "0";
    }
    // O beacon é ativado
    beacon = true;
    // A ultima leitura se torna a leitura atual
    last_change = pir_state;
  }

}

// Imprime informações para debug
void ImprimeInfo(void) {
  Serial.println("------------------------");
  Serial.print("PIR: ");
  Serial.println(pir_state);
  Serial.print("MESSAGE: ");
  Serial.println(payload);
  Serial.print("LED: ");
  Serial.println(led_state);
  Serial.println("------------------------");
}

void setup() {

  // Inicia o terminal para print de informações
  Serial.begin(115200);

  // Inicia o pino do sensor PIR em INPUT_PULLUP
  pinMode(PIR_PIN, INPUT_PULLUP);
  // Inicia o pino do led em OUTPUT
  pinMode(LAMP_PIN, OUTPUT);

  // Faz a primeira leitura do sensor PIR para iniciar o sistema de interrupção
  last_change = digitalRead(PIR_PIN);

  // Confirma que os modulos de WiFi e bluetooth estão desligados para evitar interferências indesejadas
  WiFi.mode(WIFI_OFF);
  btStop();

  // Inicia os parametros MQTT
  IniciaMQTT();

}

void loop() {

  // Imprime a leitura mais recente do sensor PIR 
  Serial.println(digitalRead(PIR_PIN));

  // Chama a função que verificao o movimento
  ChecaMovimento();

  // Caso o beacon tenha sido ativado
  if (beacon) {

    Serial.println("BEACON ACTIVATED");
    Serial.println(payload);

    // O estado do led é alterado para o estado requisitado
    digitalWrite(LAMP_PIN, led_state);

    // As conexões WiFi e MQTT são verificadas e iniciadas
    VerificaConexoes();
    
    // Envia a mensagem MQTT com o payload especificado e aguarda 1s para o envio da mesma
    MQTT.publish(TOPICO_PUBLISH_MONITORAMENTO, payload);
    MQTT.loop();
    delay(1000);

    // Desliga as conexões WiFi e MQTT para evitar interferências
    MQTT.disconnect();
    WiFi.mode(WIFI_OFF);

    // Desliga o beacon de mensagens
    beacon = false;
    Serial.println("BEACON DEACTIVATED");

    // Aguarda 10 segundos até verificar novamente
    delay(10000);

  }

  // Intervalo de 1s a cada iteração
  delay(1000);

}

```

# Hardware 

- Protoboard 400 pontos
- Jumper Linha para Protoboard
- Jumper Cabo Macho/Fêmea
- Lâmpada LED
  - Para poder representar as mudanças de estados da lâmpada de acordo comos resultados obtidos pelo sensor, utilizamos uma lâmpada LED
- Placa ESP32 Rev 1
  - A placa foi selecionada devido a sua alta velocidade de processamento eintegração nativa de módulos WI-FI e Bluetooth, facilitando assim a vconexão com os dispositivos utilizados no projeto, evitando a necessidade
de uso de módulos externos.
- Sensor de Movimento/Presença PIR - HC-SR50
  - Este módulo consiste em um sensor piroelétrico que gera energia quando exposto ao calor. Os movimentos são detectados por ele por que o corpo humano emite calor em forma de radiação infravermelha.
- Protocolo MTTQ
  - Assim como os protocolos HTTP, FTP e outros, o protocolo MTTQ foi desenvolvido para permitir a transmissão de dados entre dispositivos conectados à internet através de métodos publish/subscribe.
- Broker
  - O Broker é um serviço de provedor que intermedia a transferência dos dados entre dispositivos. Utilizamos o Mosquito, que oferece uma plataforma grátis de teste.
- Dashboard MQTT Dash
  - Para podermos visualizar as informações detectadas pelo dispositivo utilizamos um aplicativo disponível para Android chamado MQTTDash. Com ele, é possível receber e enviar as informações junto ao protocolo MQTT.
