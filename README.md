/*
============================================================
        LUXMONITOR V1.3
        LUXPOWER SNA 6K - ESP32 LAN MONITOR
============================================================

BASE READ ENGINE:
    V1.2.1 - giữ nguyên phần đọc đang hoạt động đúng

INTERFACE:
    V1.3

- READ ONLY
- Read Register 0 -> 70
- CRC16/MODBUS
- WiFi configuration via Web
- Dashboard auto update 
- PV1 + PV2 + PV3
- PV Total
- PV Current
- Battery Voltage / SOC / SOH
- Charge Current
- Discharge Current
- Grid Voltage / Frequency / Power
- Load Power
- Inverter Power
- Daily Energy
- Lifetime Energy
- Temperature
- Fault / Warning 
- Running time
============================================================
*/

#include <WiFi.h>
#include <WebServer.h>
#include <Preferences.h>
#include <time.h>
#include <WebSocketsClient.h>
#include <ArduinoJson.h>
#include <HTTPClient.h>
#include <WiFiUdp.h>

// ============================================================
// VERSION
// ============================================================

#define FW_VERSION "V1.3"

// ============================================================
// WIFI
// ============================================================

Preferences preferences;
WebServer server(80);

String wifiSSID = "";
String wifiPASS = "";

const char* AP_SSID = "LuxPower-Setup";

// ============================================================
// TIME - VIETNAM
// ============================================================

const char* NTP_SERVER = "pool.ntp.org";

const long GMT_OFFSET_SEC = 7 * 3600;
const int DAYLIGHT_OFFSET_SEC = 0;

String getUpdateTime()
{
  struct tm timeinfo;

  if (!getLocalTime(&timeinfo, 100))
  {
    return "--:--:--";
  }

  char buf[12];

  strftime(
    buf,
    sizeof(buf),
    "%H:%M:%S",
    &timeinfo
  );

  return String(buf);
}

// ============================================================
// LUXPOWER
// ============================================================

String inverterIP = "192.168.2.74";
uint16_t inverterPort = 8000;

String dongleSN = "DG51305394";
String inverterSN = "51730V3720";

WiFiClient luxClient;

// ============================================================
// REMOTE MONITOR V1.3
// ============================================================

WebSocketsClient remoteWS;

bool remoteEnable = true;
bool remoteConnected = false;

String remoteHost = "192.168.2.23";
uint16_t remotePort = 8080;
String remotePath = "/ws";

String remoteDeviceID = "DG51305394";
String remoteDeviceKey = "LUXPOWER123";

unsigned long lastRemoteSend = 0;
const unsigned long REMOTE_SEND_INTERVAL = 2000;

// Theo dõi lần cuối Remote còn kết nối
unsigned long remoteLastConnected = 0;

// Sau 10 giây không còn kết nối -> OFFLINE
const unsigned long REMOTE_OFFLINE_TIMEOUT = 10000;

// ============================================================
// OLED ESP32 - UDP
// ============================================================

WiFiUDP oledUDP;

const uint16_t OLED_UDP_PORT = 4210;
const uint16_t OLED_LOCAL_PORT = 4211;

unsigned long lastOLEDsend = 0;
const unsigned long OLED_SEND_INTERVAL = 1000;

// ============================================================
// DATA
// ============================================================

uint16_t regData[172];

bool luxConnected = false;

unsigned long lastReadOK = 0;
unsigned long lastReadAttempt = 0;

unsigned long READ_INTERVAL = 2000;  // Chu kỳ đọc LuxPower, mili giây
const unsigned long DISCONNECT_TIME = 10000;

// ============================================================
// CRC16 MODBUS
// ============================================================

uint16_t crc16_modbus(const uint8_t *data, uint16_t len)
{
  uint16_t crc = 0xFFFF;

  for (uint16_t i = 0; i < len; i++)
  {
    crc ^= data[i];

    for (uint8_t j = 0; j < 8; j++)
    {
      if (crc & 1)
        crc = (crc >> 1) ^ 0xA001;
      else
        crc >>= 1;
    }
  }

  return crc;
}

// ============================================================
// BUILD READ REQUEST
// ============================================================

int buildReadRequest(uint8_t *packet)
{
  int p = 0;

  // Header
  packet[p++] = 0xA1;
  packet[p++] = 0x1A;

  // Protocol
  packet[p++] = 0x01;
  packet[p++] = 0x00;

  // Frame length
  packet[p++] = 0x20;
  packet[p++] = 0x00;

  // Reserved
  packet[p++] = 0x01;

  // TCP function
  packet[p++] = 0xC2;

  // Dongle serial
  for (int i = 0; i < 10; i++)
  packet[p++] = dongleSN[i];

  // Data length
  packet[p++] = 0x12;
  packet[p++] = 0x00;

  // Address
  packet[p++] = 0x00;

  // Function 04
  packet[p++] = 0x04;

  // Inverter serial
  for (int i = 0; i < 10; i++)
  packet[p++] = inverterSN[i];

  // Start register
  packet[p++] = 0x00;
  packet[p++] = 0x00;

  // Count = 71
  packet[p++] = 0x47;
  packet[p++] = 0x00;

  // CRC
  uint16_t crc = crc16_modbus(
    &packet[20],
    16
  );

  packet[p++] = crc & 0xFF;
  packet[p++] = (crc >> 8) & 0xFF;

  return p;
}

// ============================================================
// READ LUXPOWER
// ============================================================

bool readLuxPower()
{
  uint8_t request[40];

  int reqLen = buildReadRequest(request);

  Serial.println();
  Serial.println("============================================");
  Serial.println("LUXPOWER READ 0-70");
  Serial.println("============================================");

  Serial.print("Request length: ");
  Serial.println(reqLen);

  // ----------------------------------------------------------
  // CONNECT
  // ----------------------------------------------------------

  if (!luxClient.connect(
      inverterIP.c_str(),
      inverterPort
    ))
  {
    Serial.println("CONNECT FAILED");
    luxClient.stop();
    return false;
  }

  luxClient.setTimeout(2000);

  // ----------------------------------------------------------
  // SEND
  // ----------------------------------------------------------

  luxClient.write(request, reqLen);

  Serial.println("REQUEST SENT");

  // ----------------------------------------------------------
  // RECEIVE
  // ----------------------------------------------------------

  uint8_t rx[1200];

  int rxLen = 0;

  unsigned long start = millis();

 while (millis() - start < 2000)
{
  // ----------------------------------------------------------
  // LOCAL WEB SERVER
  // ----------------------------------------------------------

  server.handleClient();

  // ----------------------------------------------------------
  // REMOTE WEBSOCKET
  // Cho WebSocket được xử lý liên tục
  // ----------------------------------------------------------

  if(remoteEnable)
  {
    remoteWS.loop();
  }

  // ----------------------------------------------------------
  // NHẬN DỮ LIỆU LUXPOWER
  // ----------------------------------------------------------

  while (luxClient.available())
  {
    if (rxLen < sizeof(rx))
    {
      rx[rxLen++] = luxClient.read();
    }
    else
    {
      luxClient.read();
    }
  }

  if (rxLen > 0)
  {
    delay(5);
  }

  if (rxLen >= 400)
    break;

  yield();
}

  luxClient.stop();

  Serial.print("RX bytes: ");
  Serial.println(rxLen);

  if (rxLen < 40)
  {
    Serial.println("RX TOO SHORT");
    return false;
  }

  // ----------------------------------------------------------
  // FIND FUNCTION 04 FRAME
  // ----------------------------------------------------------

  int frameStart = -1;

  for (int i = 0; i < rxLen - 25; i++)
  {
    if (rx[i] == 0xA1 &&
        rx[i + 1] == 0x1A &&
        rx[i + 7] == 0xC2 &&
        rx[i + 21] == 0x04)
    {
      frameStart = i;
      break;
    }
  }

  if (frameStart < 0)
  {
    Serial.println("FUNCTION 04 FRAME NOT FOUND");
    return false;
  }

  Serial.print("Frame 04 offset: ");
  Serial.println(frameStart);

  // ----------------------------------------------------------
  // FRAME INFORMATION
  // ----------------------------------------------------------

  int dataLengthPos = frameStart + 34;
  int dataStart = frameStart + 35;

  uint8_t valueLength = rx[dataLengthPos];

  Serial.print("Value length: ");
  Serial.println(valueLength);

  Serial.print("Data start: ");
  Serial.println(dataStart);

  // 172 registers = 0xAC bytes
  if (valueLength < 142)
  {
    Serial.println("NOT ENOUGH DATA");
    return false;
  }

  if (dataStart + 142 > rxLen)
  {
    Serial.println("DATA OUT OF RANGE");
    return false;
  }

  // ----------------------------------------------------------
  // PARSE REGISTERS
  // ----------------------------------------------------------

  for (int i = 0; i < 172; i++)
  {
    int pos = dataStart + i * 2;

    regData[i] =
      (uint16_t)rx[pos] |
      ((uint16_t)rx[pos + 1] << 8);
  }

  Serial.println("READ OK");
  Serial.print("REG16 = ");
  Serial.println((int16_t)regData[16]);

  Serial.print("REG24 = ");
  Serial.println((int16_t)regData[24]);

  Serial.print("REG26 = ");
  Serial.println((int16_t)regData[26]);

  Serial.print("REG27 = ");
  Serial.println((int16_t)regData[27]);

  lastReadOK = millis();
  luxConnected = true;

  return true;

  Serial.print("REG31 + GRID IMPORT = ");
  Serial.print(regData[31] / 10.0 + gridImportDay(), 1);
  Serial.println(" kWh");
}
// ============================================================
// READ CONSUMPTION TODAY - REG171
// READ ONLY
// ============================================================

bool readConsumptionDay()
{
  uint8_t request[40];
  int p = 0;

  // =========================================================
  // READ REG170 -> REG173
  // REG170 = Pload
  // REG171 = Eload_day
  // REG172 = Eload_allL
  // REG173 = Eload_allH
  // =========================================================

  request[p++] = 0xA1;
  request[p++] = 0x1A;

  request[p++] = 0x01;
  request[p++] = 0x00;

  request[p++] = 0x20;
  request[p++] = 0x00;

  request[p++] = 0x01;
  request[p++] = 0xC2;

  // Dongle SN
  for (int i = 0; i < 10; i++)
  request[p++] = dongleSN[i];

  // Data length
  request[p++] = 0x12;
  request[p++] = 0x00;

  // Address
  request[p++] = 0x00;

  // Function 04
  request[p++] = 0x04;

  // Inverter SN
  for (int i = 0; i < 10; i++)
  request[p++] = inverterSN[i];

  // Start register = 170
  request[p++] = 0xAA;
  request[p++] = 0x00;

  // Quantity = 4
  request[p++] = 0x04;
  request[p++] = 0x00;

  // CRC
  uint16_t crc = crc16_modbus(&request[20], 16);

  request[p++] = crc & 0xFF;
  request[p++] = crc >> 8;

  // =========================================================
  // CONNECT
  // =========================================================

  WiFiClient client;

  if (!client.connect(
      inverterIP.c_str(),
      inverterPort
    ))
  {
    Serial.println("CONSUMPTION READ: CONNECT FAILED");
    return false;
  }

  // =========================================================
  // SEND
  // =========================================================

  client.write(request, p);
  client.flush();

  // =========================================================
  // RECEIVE
  // =========================================================

  uint8_t rx[1200];
  int rxLen = 0;

  unsigned long startTime = millis();

  while (millis() - startTime < 2000)
  {
    while (client.available())
    {
      if (rxLen < sizeof(rx))
      {
        rx[rxLen++] = client.read();
      }
      else
      {
        client.read();
      }
    }

    delay(5);
  }

  client.stop();

  if (rxLen < 43)
  {
    Serial.println("CONSUMPTION READ: RX TOO SHORT");
    return false;
  }

  // =========================================================
  // SEARCH FRAME START REG = 170
  // =========================================================

  for (int i = 0; i <= rxLen - 43; i++)
  {
    if (rx[i] != 0xA1)
      continue;

    if (rx[i + 1] != 0x1A)
      continue;

    if (rx[i + 7] != 0xC2)
      continue;

    // Function = 04
    if (rx[i + 21] != 0x04)
      continue;

    // Start register
    uint16_t startReg =
      rx[i + 32] |
      ((uint16_t)rx[i + 33] << 8);

    if (startReg != 170)
      continue;

    // =======================================================
    // VALUE LENGTH
    // =======================================================

    uint8_t valueLength = rx[i + 34];

    if (valueLength < 8)
      continue;

    if (i + 43 > rxLen)
      continue;

    // =======================================================
    // READ REG170
    // =======================================================

    uint16_t reg170 =
      rx[i + 35] |
      ((uint16_t)rx[i + 36] << 8);

    // =======================================================
    // READ REG171
    // =======================================================

    uint16_t reg171 =
      rx[i + 37] |
      ((uint16_t)rx[i + 38] << 8);

    // =======================================================
    // READ REG172
    // =======================================================

    uint16_t reg172 =
      rx[i + 39] |
      ((uint16_t)rx[i + 40] << 8);

    // =======================================================
    // READ REG173
    // =======================================================

    uint16_t reg173 =
      rx[i + 41] |
      ((uint16_t)rx[i + 42] << 8);

    // =======================================================
    // STORE ONLY REG171
    // =======================================================

    regData[171] = reg171;

    // =======================================================
    // DEBUG SHORT
    // =======================================================

    Serial.print("CONSUMPTION: REG171 = ");
    Serial.print(reg171);
    Serial.print(" -> ");
    Serial.print(reg171 / 10.0, 1);
    Serial.println(" kWh");

    return true;
  }

  Serial.println("CONSUMPTION READ: REG170 FRAME NOT FOUND");

  return false;
}

// ============================================================
// BASIC DATA
// ============================================================

float pv1Voltage()
{
  return regData[1] / 10.0;
}

float pv2Voltage()
{
  return regData[2] / 10.0;
}

float batteryVoltage()
{
  return regData[4] / 10.0;
}

int batterySOC()
{
  return regData[5] & 0xFF;
}

int batterySOH()
{
  return (regData[5] >> 8) & 0xFF;
}

int pv1Power()
{
  return regData[7];
}

int pv2Power()
{
  return regData[8];
}

int pv3Power()
{
  return regData[9];
}

int pvTotal()
{
  return pv1Power() +
         pv2Power() +
         pv3Power();
}

int chargePower()
{
  return regData[10];
}

int dischargePower()
{
  return regData[11];
}

float gridVoltage()
{
  return regData[12] / 10.0;
}

float gridFrequency()
{
  return regData[15] / 100.0;
}

int inverterPower()
{
  return regData[16];
}

int gridPower()
{
  return regData[27];
}

int loadPower()
{
  return inverterPower() + gridPower();
}

int epsPower()
{
  return regData[24];
}

// ============================================================
// CURRENT
// ============================================================

float pv1Current()
{
  float v = pv1Voltage();
  float p = pv1Power();

  if (v <= 1.0 || p <= 0)
    return 0;

  return p / v;
}

float pv2Current()
{
  float v = pv2Voltage();
  float p = pv2Power();

  if (v <= 1.0 || p <= 0)
    return 0;

  return p / v;
}

float pvTotalCurrent()
{
  return pv1Current() + pv2Current();
}

float chargeCurrent()
{
  float v = batteryVoltage();
  float p = chargePower();

  if (v <= 1.0 || p <= 0)
    return 0;

  return p / v;
}

float dischargeCurrent()
{
  float v = batteryVoltage();
  float p = dischargePower();

  if (v <= 1.0 || p <= 0)
    return 0;

  return p / v;
}

// ============================================================
// DAILY ENERGY
// ============================================================

float pvDay()
{
  return (
    regData[28] +
    regData[29] +
    regData[30]
  ) / 10.0;
}

float chargeDay()
{
  return regData[33] / 10.0;
}

float dischargeDay()
{
  return regData[34] / 10.0;
}

float exportDay()
{
  // REG36 = Đẩy lưới hôm nay
  return regData[36] / 10.0;
}

float gridImportDay()
{
  // REG37 = Lấy lưới hôm nay
  return regData[37] / 10.0;
}

float consumptionDay()
{
  return (regData[31] + regData[35] + regData[37]) / 10.0;
}

float loadDay()
{
  return regData[37] / 10.0;
}

// ============================================================
// 32 BIT
// ============================================================

uint32_t make32(uint16_t low, uint16_t high)
{
  return ((uint32_t)high << 16) | low;
}

// ============================================================
// LIFETIME ENERGY
// ============================================================

float pvAll()
{
  uint32_t pv1 = make32(regData[40], regData[41]);
  uint32_t pv2 = make32(regData[42], regData[43]);
  uint32_t pv3 = make32(regData[44], regData[45]);

  return (pv1 + pv2 + pv3) / 10.0;
}

float chargeAll()
{
  uint32_t v = make32(regData[50], regData[51]);
  return v / 10.0;
}

float dischargeAll()
{
  uint32_t v = make32(regData[52], regData[53]);
  return v / 10.0;
}

float epsAll()
{
  uint32_t v = make32(regData[54], regData[55]);
  return v / 10.0;
}

float exportAll()
{
  uint32_t v = make32(regData[56], regData[57]);
  return v / 10.0;
}

float gridImportAll()
{
  uint32_t v = make32(regData[58], regData[59]);
  return v / 10.0;
}

float consumptionAll()
{
  uint32_t inv = make32(regData[46], regData[47]);
  uint32_t eps = make32(regData[54], regData[55]);
  uint32_t grid = make32(regData[58], regData[59]);

  return (inv + eps + grid) / 10.0;
}

// ============================================================
// DIAGNOSTIC
// ============================================================

uint32_t faultCode()
{
  return make32(regData[60], regData[61]);
}

uint32_t warningCode()
{
  return make32(regData[62], regData[63]);
}

// ============================================================
// TEMPERATURE
// ============================================================

int inverterTemp()
{
  return (int16_t)regData[64];
}

int acTemp()
{
  return (int16_t)regData[65];
}

int dcTemp()
{
  return (int16_t)regData[66];
}

int batteryTemp()
{
  return (int16_t)regData[67];
}

// ============================================================
// RUNNING TIME
// ============================================================

uint32_t runningTimeSeconds()
{
  return make32(regData[69], regData[70]);
}

String runningTimeText()
{
  uint32_t totalSeconds = runningTimeSeconds();

  uint32_t days = totalSeconds / 86400;
  totalSeconds %= 86400;

  uint32_t hours = totalSeconds / 3600;
  totalSeconds %= 3600;

  uint32_t minutes = totalSeconds / 60;
  uint32_t seconds = totalSeconds % 60;

  String s = "";

  if (days > 0)
  {
    s += String(days);
    s += " ngày ";
  }

  if (hours > 0 || days > 0)
  {
    s += String(hours);
    s += " giờ ";
  }

  s += String(minutes);
  s += " phút";

  return s;
}

// ============================================================
// REMOTE JSON DATA
// ============================================================

String buildRemoteJSON()
{
  StaticJsonDocument<2048> doc;

  // ----------------------------------------------------------
  // DEVICE
  // ----------------------------------------------------------
  doc["device"]  = remoteDeviceID;
  doc["key"]     = remoteDeviceKey;
  doc["version"] = FW_VERSION;
  doc["online"]  = luxConnected;

  // ----------------------------------------------------------
  // CURRENT
  // ----------------------------------------------------------
  doc["pv1_voltage"]     = pv1Voltage();
  doc["pv2_voltage"]     = pv2Voltage();
  doc["battery_voltage"] = batteryVoltage();
  doc["battery_soc"]     = batterySOC();
  doc["battery_soh"]     = batterySOH();

  doc["pv1_power"]       = pv1Power();
  doc["pv2_power"]       = pv2Power();
  doc["pv3_power"]       = pv3Power();
  doc["pv_total"]        = pvTotal();

  doc["charge_power"]    = chargePower();
  doc["discharge_power"] = dischargePower();

  doc["grid_voltage"]    = gridVoltage();
  doc["grid_frequency"]  = gridFrequency();
  doc["inverter_power"]  = inverterPower();
  doc["grid_power"]      = gridPower();
  doc["load_power"]      = loadPower();
  doc["eps_power"]       = epsPower();

  // ----------------------------------------------------------
  // TODAY
  // ----------------------------------------------------------
  doc["pv_day"]          = pvDay();
  doc["charge_day"]      = chargeDay();
  doc["discharge_day"]   = dischargeDay();
  doc["grid_import_day"] = gridImportDay();
  doc["export_day"]      = exportDay();
  doc["consumption_day"] = consumptionDay();

  // ----------------------------------------------------------
  // LIFETIME
  // ----------------------------------------------------------
  doc["pv_all"]           = pvAll();
  doc["charge_all"]       = chargeAll();
  doc["discharge_all"]    = dischargeAll();
  doc["grid_import_all"]  = gridImportAll();
  doc["export_all"]       = exportAll();
  doc["consumption_all"]  = consumptionAll();

  // ----------------------------------------------------------
  // TEMPERATURE
  // ----------------------------------------------------------
  doc["inverter_temp"] = inverterTemp();
  doc["ac_temp"]       = acTemp();
  doc["dc_temp"]       = dcTemp();
  doc["battery_temp"]  = batteryTemp();

  // ----------------------------------------------------------
  // DIAGNOSTIC
  // ----------------------------------------------------------
  doc["fault"]   = faultCode();
  doc["warning"] = warningCode();

  // ----------------------------------------------------------
  // RUNTIME
  // ----------------------------------------------------------
  doc["runtime"] = runningTimeSeconds();

  // ----------------------------------------------------------
  // INVERTER INFO
  // ----------------------------------------------------------
  doc["inverter_ip"]   = inverterIP;
  doc["inverter_port"] = inverterPort;
  doc["dongle"]        = dongleSN;
  doc["inverter_sn"]   = inverterSN;

  // ----------------------------------------------------------
  // TIMESTAMP
  // ----------------------------------------------------------
  doc["millis"] = millis();

  String output;
  serializeJson(doc, output);

  return output;
}

// ============================================================
// SEND DATA TO OLED ESP32
// ============================================================

void sendOLEDData()
{
  if(WiFi.status() != WL_CONNECTED)
    return;

  if(millis() - lastOLEDsend < OLED_SEND_INTERVAL)
    return;

  lastOLEDsend = millis();

  String json = buildRemoteJSON();

  IPAddress oledBroadcast(
    192, 168, 2, 255
  );

  oledUDP.beginPacket(
    oledBroadcast,
    OLED_UDP_PORT
  );

  oledUDP.print(json);

  oledUDP.endPacket();

  Serial.print("OLED UDP TX: ");
  Serial.println(json);
}

// ============================================================
// REMOTE WEBSOCKET EVENT
// ============================================================

void remoteWebSocketEvent(WStype_t type,
                          uint8_t * payload,
                          size_t length)
{
  switch(type)
  {
   case WStype_DISCONNECTED:

  remoteConnected = false;

  Serial.println();
  Serial.println("============================================");
  Serial.println("REMOTE: DISCONNECTED");
  Serial.println("============================================");

  break;

    case WStype_CONNECTED:

  remoteConnected = true;
  remoteLastConnected = millis();

  Serial.println();
  Serial.println("============================================");
  Serial.println("REMOTE: CONNECTED");
  Serial.println("============================================");

  break;

    case WStype_TEXT:
      Serial.print("REMOTE RX: ");

      for(size_t i = 0; i < length; i++)
      {
        Serial.print((char)payload[i]);
      }

      Serial.println();
      break;

    case WStype_ERROR:
      remoteConnected = false;
      Serial.println();
      Serial.println("REMOTE: ERROR");
      break;

    default:
      break;
  }
}

// ============================================================
// START REMOTE
// ============================================================

void startRemote()
{
  if(!remoteEnable)
  {
    Serial.println("REMOTE: DISABLED");
    return;
  }

  if(remoteHost.length() == 0)
  {
    Serial.println("REMOTE: SERVER NOT CONFIGURED");
    return;
  }

  Serial.println();
  Serial.println("============================================");
  Serial.println("REMOTE MONITOR V1.3");
  Serial.println("============================================");

  Serial.print("SERVER: ");
  Serial.println(remoteHost);

  Serial.print("PORT: ");
  Serial.println(remotePort);

  Serial.print("PATH: ");
  Serial.println(remotePath);

  Serial.print("DEVICE: ");
  Serial.println(remoteDeviceID);

  remoteWS.begin(remoteHost.c_str(),
               remotePort,
               remotePath.c_str(),
               "arduino");

  remoteWS.onEvent(remoteWebSocketEvent);

  remoteWS.setReconnectInterval(1000);

  Serial.println("REMOTE STARTED");
}

// ============================================================
// SEND REMOTE DATA
// ============================================================

void sendRemoteData()
{
  if(!remoteEnable)
    return;

  if(!remoteConnected)
    return;

  if(millis() - lastRemoteSend < REMOTE_SEND_INTERVAL)
    return;

  lastRemoteSend = millis();

  remoteLastConnected = millis();

  String json = buildRemoteJSON();

  remoteWS.sendTXT(json);

  Serial.print("REMOTE TX: ");
  Serial.println(json);
}

bool isRemoteOnline()
{
  if(!remoteEnable)
    return false;

  if(!remoteConnected)
    return false;

  if(millis() - remoteLastConnected > REMOTE_OFFLINE_TIMEOUT)
    return false;

  return true;
}

// ============================================================
// WIFI
// ============================================================

void loadWiFi()
{
  preferences.begin("wifi", true);

  wifiSSID = preferences.getString("ssid", "");
  wifiPASS = preferences.getString("pass", "");

  preferences.end();
}

void saveWiFi(String ssid, String pass)
{
  preferences.begin("wifi", false);

  preferences.putString("ssid", ssid);
  preferences.putString("pass", pass);

  preferences.end();
}

void clearWiFi()
{
  preferences.begin("wifi", false);

  preferences.clear();

  preferences.end();
}

// ============================================================
// READ INTERVAL SETTINGS
// Cho phép chỉnh chu kỳ đọc LuxPower từ 2 đến 30 giây
// ============================================================

void loadReadInterval()
{
  preferences.begin("readcfg", true);
  uint32_t seconds = preferences.getUInt("seconds", 2);
  preferences.end();

  if (seconds < 2 || seconds > 30)
    seconds = 2;

  READ_INTERVAL = seconds * 1000UL;
}

void handleSaveReadInterval()
{
  String secondsText = server.arg("seconds");
  secondsText.trim();

  if (secondsText.length() == 0)
  {
    server.send(
      400,
      "text/html; charset=utf-8",
      "<meta charset='UTF-8'><h2>Vui lòng nhập thời gian từ 2 đến 30 giây.</h2><a href='/'>Quay lại Dashboard</a>"
    );
    return;
  }

  long seconds = secondsText.toInt();

  if (seconds < 2 || seconds > 30)
  {
    server.send(
      400,
      "text/html; charset=utf-8",
      "<meta charset='UTF-8'><h2>Thời gian phải nằm trong khoảng 2 đến 30 giây.</h2><a href='/'>Quay lại Dashboard</a>"
    );
    return;
  }

  preferences.begin("readcfg", false);
  preferences.putUInt("seconds", (uint32_t)seconds);
  preferences.end();

  READ_INTERVAL = (unsigned long)seconds * 1000UL;
  lastReadAttempt = millis();

  server.sendHeader("Location", "/", true);
  server.send(303, "text/plain; charset=utf-8", "Đã lưu chu kỳ đọc LuxPower.");
}

// ============================================================
// INVERTER CONFIG
// ============================================================

void loadInverter()
{
  preferences.begin("inverter", true);

  inverterIP =
    preferences.getString(
      "ip",
      "192.168.2.74"
    );

  inverterPort =
    preferences.getUShort(
      "port",
      8000
    );

  dongleSN =
    preferences.getString(
      "dongle",
      "DG51305394"
    );

  inverterSN =
    preferences.getString(
      "inv",
      "51730V3720"
    );

  preferences.end();
}

void saveInverter(
  String ip,
  uint16_t port,
  String dongle,
  String inv
)
{
  preferences.begin("inverter", false);

  preferences.putString("ip", ip);
  preferences.putUShort("port", port);
  preferences.putString("dongle", dongle);
  preferences.putString("inv", inv);

  preferences.end();
}

// ============================================================
// CONNECT WIFI
// ============================================================

void connectWiFi()
{
  loadWiFi();

  if (wifiSSID.length() == 0)
  {
    WiFi.mode(WIFI_AP);

    WiFi.softAP(AP_SSID);

    Serial.println();
    Serial.println("=================================");
    Serial.println(" WIFI SETUP MODE");
    Serial.println("=================================");

    Serial.print("AP: ");
    Serial.println(AP_SSID);

    Serial.print("IP: ");
    Serial.println(WiFi.softAPIP());

    return;
  }

  WiFi.mode(WIFI_STA);

  WiFi.begin(
    wifiSSID.c_str(),
    wifiPASS.c_str()
  );

  Serial.println();
  Serial.println("=================================");
  Serial.println(" CONNECTING WIFI");
  Serial.println("=================================");

  Serial.print("SSID: ");
  Serial.println(wifiSSID);

  unsigned long start = millis();

  while (
    WiFi.status() != WL_CONNECTED &&
    millis() - start < 15000
  )
  {
    delay(500);
    Serial.print(".");
  }

  Serial.println();

  if (WiFi.status() == WL_CONNECTED)
{
  Serial.println("WiFi OK");

  Serial.print("ESP32 IP: ");
  Serial.println(WiFi.localIP());

  // ----------------------------------------------------------
  // NTP TIME - VIETNAM UTC+7
  // ----------------------------------------------------------

  configTime(
    GMT_OFFSET_SEC,
    DAYLIGHT_OFFSET_SEC,
    NTP_SERVER
  );

  Serial.println("Synchronizing time...");

  struct tm timeinfo;

  if (getLocalTime(&timeinfo, 5000))
  {
    Serial.println("TIME OK");

    Serial.print("ESP32 TIME: ");

    Serial.printf(
      "%02d:%02d:%02d\n",
      timeinfo.tm_hour,
      timeinfo.tm_min,
      timeinfo.tm_sec
    );
  }
  else
  {
    Serial.println("TIME SYNC FAILED");
  }
}
  else
  {
    Serial.println("WiFi FAILED");
    Serial.println("Starting AP...");

    WiFi.mode(WIFI_AP);

    WiFi.softAP(AP_SSID);

    Serial.print("AP IP: ");
    Serial.println(WiFi.softAPIP());
  }
}

// ============================================================
// API - V1.3 FIX
// ============================================================

void handleAPI()
{
  String json;
  json.reserve(5000);

  unsigned long age = 0;

  if (lastReadOK > 0)
    age = millis() - lastReadOK;

  json += "{";

  // STATUS
  json += "\"version\":\"";
  json += FW_VERSION;
  json += "\",";

  json += "\"online\":";
  json += (luxConnected ? "true" : "false");
  json += ",";

  json += "\"age\":";
  json += String(age);
  json += ",";

  json += "\"updatetime\":\"";
  json += getUpdateTime();
  json += "\",";

  // ----------------------------------------------------------
  // PV
  // ----------------------------------------------------------

  json += "\"pv1v\":";
  json += String(pv1Voltage(), 1);
  json += ",";

  json += "\"pv1w\":";
  json += String(pv1Power());
  json += ",";

  json += "\"pv1a\":";
  json += String(pv1Current(), 1);
  json += ",";

  json += "\"pv2v\":";
  json += String(pv2Voltage(), 1);
  json += ",";

  json += "\"pv2w\":";
  json += String(pv2Power());
  json += ",";

  json += "\"pv2a\":";
  json += String(pv2Current(), 1);
  json += ",";

  json += "\"pv3w\":";
  json += String(pv3Power());
  json += ",";

  json += "\"pvt\":";
  json += String(pvTotal());
  json += ",";

  json += "\"pva\":";
  json += String(pvTotalCurrent(), 1);
  json += ",";

  // ----------------------------------------------------------
  // BATTERY
  // ----------------------------------------------------------

  json += "\"battv\":";
  json += String(batteryVoltage(), 1);
  json += ",";

  json += "\"soc\":";
  json += String(batterySOC());
  json += ",";

  json += "\"soh\":";
  json += String(batterySOH());
  json += ",";

  json += "\"chargew\":";
  json += String(chargePower());
  json += ",";

  json += "\"chargea\":";
  json += String(chargeCurrent(), 1);
  json += ",";

  json += "\"dischargew\":";
  json += String(dischargePower());
  json += ",";

  json += "\"dischargea\":";
  json += String(dischargeCurrent(), 1);
  json += ",";

  // ----------------------------------------------------------
  // GRID
  // ----------------------------------------------------------

  json += "\"gridv\":";
  json += String(gridVoltage(), 1);
  json += ",";

  json += "\"gridhz\":";
  json += String(gridFrequency(), 2);
  json += ",";

  json += "\"gridw\":";
  json += String(gridPower());
  json += ",";

  json += "\"exportw\":";
  json += String(regData[26]);
  json += ",";

  // ----------------------------------------------------------
  // LOAD / INVERTER
  // ----------------------------------------------------------

  json += "\"loadw\":";
  json += String(loadPower());
  json += ",";

  json += "\"invw\":";
  json += String(inverterPower());
  json += ",";

  json += "\"epsw\":";
  json += String(epsPower());
  json += ",";

  // ----------------------------------------------------------
  // DAILY
  // ----------------------------------------------------------

  json += "\"pvday\":";
  json += String(pvDay(), 1);
  json += ",";

  json += "\"chargeday\":";
  json += String(chargeDay(), 1);
  json += ",";

  json += "\"dischargeday\":";
  json += String(dischargeDay(), 1);
  json += ",";

  json += "\"griday\":";
  json += String(gridImportDay(), 1);
  json += ",";

  json += "\"exportday\":";
  json += String(exportDay(), 1);
  json += ",";

  json += "\"consumptionday\":";
  json += String(consumptionDay(), 1);
  json += ",";

  // ----------------------------------------------------------
  // LIFETIME
  // ----------------------------------------------------------

  json += "\"pvall\":";
json += String(pvAll(), 1);
json += ",";

json += "\"consumptionall\":";
json += String(consumptionAll(), 1);
json += ",";

json += "\"chargeall\":";
json += String(chargeAll(), 1);
json += ",";

json += "\"dischargeall\":";
json += String(dischargeAll(), 1);
json += ",";

json += "\"exportall\":";
json += String(exportAll(), 1);
json += ",";

json += "\"gridimportall\":";
json += String(gridImportAll(), 1);
json += ",";

  // ----------------------------------------------------------
  // TEMPERATURE
  // ----------------------------------------------------------

  json += "\"invt\":";
  json += String(inverterTemp());
  json += ",";

  json += "\"act\":";
  json += String(acTemp());
  json += ",";

  json += "\"dct\":";
  json += String(dcTemp());
  json += ",";

  json += "\"battt\":";
  json += String(batteryTemp());
  json += ",";

  // ----------------------------------------------------------
  // DIAGNOSTIC
  // ----------------------------------------------------------

  json += "\"fault\":";
  json += String(faultCode());
  json += ",";

  json += "\"warning\":";
  json += String(warningCode());
  json += ",";

  json += "\"runtime\":\"";
  json += runningTimeText();
  json += "\",";

  // ----------------------------------------------------------
  // IP
  // ----------------------------------------------------------

  json += "\"ip\":\"";

  if (WiFi.getMode() == WIFI_AP)
    json += WiFi.softAPIP().toString();
  else
    json += WiFi.localIP().toString();

  json += "\"";

  json += "}";

  // ----------------------------------------------------------
  // NO CACHE
  // ----------------------------------------------------------

  server.sendHeader(
    "Cache-Control",
    "no-cache, no-store, must-revalidate"
  );

  server.sendHeader(
    "Pragma",
    "no-cache"
  );

  server.sendHeader(
    "Expires",
    "0"
  );

  server.send(
    200,
    "application/json; charset=utf-8",
    json
  );
}

// ============================================================
// WIFI PAGE
// ============================================================

void handleWiFi()
{
  String page;

  page.reserve(5000);

  page += R"rawliteral(
<!DOCTYPE html>
<html>

<head>

<meta charset="UTF-8">

<meta name="viewport"
content="width=device-width,initial-scale=1">

<title>LuxMonitor WiFi</title>

<style>

body{
  margin:0;
  font-family:Arial,sans-serif;
  background:#101318;
  color:#fff;
  padding:20px;
}

.box{
  max-width:450px;
  margin:30px auto;
  background:#1a1f27;
  border-radius:18px;
  padding:25px;
  box-sizing:border-box;
}

h1{
  margin-top:0;
  text-align:center;
}

label{
  display:block;
  margin-top:15px;
  margin-bottom:6px;
  color:#aeb7c5;
}

input{
  width:100%;
  box-sizing:border-box;
  padding:13px;
  border:0;
  border-radius:10px;
  background:#2a303a;
  color:white;
  font-size:16px;
}

button{
  width:100%;
  padding:14px;
  margin-top:20px;
  border:0;
  border-radius:10px;
  background:#18b77a;
  color:white;
  font-size:16px;
  font-weight:bold;
}

.back{
  display:block;
  text-align:center;
  margin-top:20px;
  color:#7ec8ff;
  text-decoration:none;
}

.header-actions{
 display:flex;
 align-items:center;
 gap:8px;
 flex-wrap:wrap;
 justify-content:flex-end;
}

.remote-btn{
 display:inline-block;
 text-decoration:none;
 color:#fff;
 background:#1d6fa5;
 padding:9px 13px;
 border-radius:9px;
 font-size:13px;
 font-weight:bold;
 border:0;
 cursor:pointer;
 white-space:nowrap;
}

.remote-btn:hover{
 background:#2789c7;
}

</style>

</head>

<body>

<div class="box">

<h1>⚡ LuxMonitor</h1>

<form action="/savewifi" method="POST">

<label>WiFi SSID</label>

<input
name="ssid"
value=")rawliteral";

  page += wifiSSID;

  page += R"rawliteral(">

<label>Mật khẩu WiFi</label>

<input
type="password"
name="pass"
value=")rawliteral";

  page += wifiPASS;

  page += R"rawliteral(">

<button type="submit">
💾 Lưu WiFi & Khởi động lại
</button>

</form>

<a class="back" href="/">
← Quay lại Dashboard
</a>

</div>

</body>
</html>
)rawliteral";

  server.send(
    200,
    "text/html; charset=utf-8",
    page
  );
}

// ============================================================
// INVERTER PAGE
// ============================================================

void handleInverter()
{
  String page;

  page.reserve(7000);

  page += R"rawliteral(
<!DOCTYPE html>
<html>

<head>

<meta charset="UTF-8">

<meta name="viewport"
content="width=device-width,initial-scale=1">

<title>LuxMonitor - Cài đặt biến tần</title>

<style>

body{
  margin:0;
  font-family:Arial,sans-serif;
  background:#101318;
  color:#fff;
  padding:20px;
}

.box{
  max-width:450px;
  margin:30px auto;
  background:#1a1f27;
  border-radius:18px;
  padding:25px;
  box-sizing:border-box;
}

h1{
  margin-top:0;
  text-align:center;
}

label{
  display:block;
  margin-top:15px;
  margin-bottom:6px;
  color:#aeb7c5;
}

input{
  width:100%;
  box-sizing:border-box;
  padding:13px;
  border:0;
  border-radius:10px;
  background:#2a303a;
  color:white;
  font-size:16px;
}

button{
  width:100%;
  padding:14px;
  margin-top:20px;
  border:0;
  border-radius:10px;
  background:#18b77a;
  color:white;
  font-size:16px;
  font-weight:bold;
}

.back{
  display:block;
  text-align:center;
  margin-top:20px;
  color:#7ec8ff;
  text-decoration:none;
}

.info{
  margin-top:20px;
  padding:12px;
  background:#222832;
  border-radius:10px;
  color:#9da8b7;
  font-size:13px;
  line-height:1.6;
}

</style>

</head>

<body>

<div class="box">

<h1>⚡ Biến tần LuxPower</h1>

<form action="/saveinverter" method="POST">

<label>IP biến tần</label>

<input
name="ip"
value=")rawliteral";

  page += inverterIP;

  page += R"rawliteral("
placeholder="192.168.2.74">

<label>Port</label>

<input
type="number"
name="port"
value=")rawliteral";

  page += String(inverterPort);

  page += R"rawliteral("
placeholder="8000">

<label>Dongle SN</label>

<input
name="dongle"
value=")rawliteral";

  page += dongleSN;

  page += R"rawliteral("
maxlength="10">

<label>Inverter SN</label>

<input
name="inv"
value=")rawliteral";

  page += inverterSN;

  page += R"rawliteral("
maxlength="10">

<button type="submit">
💾 Lưu cấu hình & Khởi động lại
</button>

</form>

<div class="info">

IP: )rawliteral";

  page += inverterIP;

  page += R"rawliteral(<br>

Port: )rawliteral";

  page += String(inverterPort);

  page += R"rawliteral(<br>

Dongle: )rawliteral";

  page += dongleSN;

  page += R"rawliteral(<br>

Inverter: )rawliteral";

  page += inverterSN;

  page += R"rawliteral(

</div>

<a class="back" href="/">
← Quay lại Dashboard
</a>

</div>

</body>

</html>
)rawliteral";

  server.send(
    200,
    "text/html; charset=utf-8",
    page
  );
}

// ============================================================
// SAVE WIFI
// ============================================================

void handleSaveWiFi()
{
  String ssid = server.arg("ssid");
  String pass = server.arg("pass");

  if (ssid.length() == 0)
  {
    server.send(
      400,
      "text/html",
      "SSID empty"
    );

    return;
  }

  saveWiFi(ssid, pass);

  String page;

  page += "<html>";
  page += "<meta charset='UTF-8'>";
  page += "<body style='font-family:Arial;text-align:center;padding:50px'>";
  page += "<h2>Đã lưu WiFi</h2>";
  page += "<p>ESP32 đang khởi động lại...</p>";
  page += "</body></html>";

  server.send(
    200,
    "text/html; charset=utf-8",
    page
  );

  delay(1000);

  ESP.restart();
}

// ============================================================
// SAVE INVERTER
// ============================================================

void handleSaveInverter()
{
  String ip = server.arg("ip");
  String portText = server.arg("port");
  String dongle = server.arg("dongle");
  String inv = server.arg("inv");

  if (
    ip.length() == 0 ||
    portText.length() == 0 ||
    dongle.length() != 10 ||
    inv.length() != 10
  )
  {
    server.send(
      400,
      "text/html; charset=utf-8",
      "<h2>Dữ liệu không hợp lệ</h2>"
    );

    return;
  }

  uint16_t port =
    (uint16_t)portText.toInt();

  if (port == 0)
    port = 8000;

  saveInverter(
    ip,
    port,
    dongle,
    inv
  );

  String page;

  page += "<html>";
  page += "<meta charset='UTF-8'>";
  page += "<body style='font-family:Arial;text-align:center;padding:50px'>";
  page += "<h2>Đã lưu cấu hình biến tần</h2>";
  page += "<p>ESP32 đang khởi động lại...</p>";
  page += "</body></html>";

  server.send(
    200,
    "text/html; charset=utf-8",
    page
  );

  delay(1000);

  ESP.restart();
}

// ============================================================
// CLEAR WIFI
// ============================================================

void handleClearWiFi()
{
  clearWiFi();

  server.send(
    200,
    "text/html; charset=utf-8",
    "<h2>WiFi da xoa. ESP32 khoi dong lai...</h2>"
  );

  delay(1000);

  ESP.restart();
}

// ============================================================
// REMOTE URL
// ESP32 -> PC SERVER -> CLOUDFLARE URL
// ============================================================

void handleRemote()
{
  WiFiClient remoteClient;
  HTTPClient http;

  String url =
    "http://" +
    remoteHost +
    ":" +
    String(remotePort) +
    "/remote?t=" +
    String(millis());

  Serial.println();
  Serial.println("============================================");
  Serial.println("REMOTE URL REQUEST");
  Serial.println("============================================");
  Serial.print("PC SERVER: ");
  Serial.println(url);

  http.setTimeout(3000);

  if (!http.begin(remoteClient, url))
  {
    Serial.println("HTTP BEGIN FAILED");

    server.send(
      500,
      "text/html; charset=utf-8",
      "<html><meta charset='UTF-8'><body style='font-family:Arial;text-align:center;padding:50px'><h2>Không kết nối được PC</h2><p>Không thể tạo kết nối tới PC Server.</p><a href='/'>← Quay lại Dashboard</a></body></html>"
    );

    return;
  }

  int httpCode = http.GET();

  Serial.print("HTTP CODE: ");
  Serial.println(httpCode);

  if (httpCode == HTTP_CODE_OK)
  {
    String response = http.getString();

    Serial.print("REMOTE RESPONSE: ");
    Serial.println(response);

    http.end();

    StaticJsonDocument<512> doc;

    DeserializationError error =
      deserializeJson(doc, response);

    if (error)
    {
      Serial.println("REMOTE URL JSON ERROR");

      server.send(
        500,
        "text/html; charset=utf-8",
        "<html><meta charset='UTF-8'><body style='font-family:Arial;text-align:center;padding:50px'><h2>Lỗi dữ liệu Remote</h2><p>PC Server trả về dữ liệu không hợp lệ.</p><a href='/'>← Quay lại Dashboard</a></body></html>"
      );

      return;
    }

    String remoteURL = doc["url"] | "";

    remoteURL.trim();

    Serial.print("CLOUDFLARE URL: ");
    Serial.println(remoteURL);

    if (remoteURL.length() == 0)
    {
      server.send(
        200,
        "text/html; charset=utf-8",
        "<html><meta charset='UTF-8'><body style='font-family:Arial;text-align:center;padding:50px'><h2>Chưa có URL Remote</h2><p>Hãy chạy Cloudflare Tunnel trên máy tính trước.</p><a href='/'>← Quay lại Dashboard</a></body></html>"
      );

      return;
    }

    if (!remoteURL.startsWith("https://"))
    {
      server.send(
        500,
        "text/html; charset=utf-8",
        "<html><meta charset='UTF-8'><body style='font-family:Arial;text-align:center;padding:50px'><h2>URL Remote không hợp lệ</h2><a href='/'>← Quay lại Dashboard</a></body></html>"
      );

      return;
    }

    // --------------------------------------------------------
    // CHUYỂN TRÌNH DUYỆT SANG CLOUDFLARE
    // --------------------------------------------------------

    server.sendHeader(
      "Location",
      remoteURL,
      true
    );

    server.send(
      302,
      "text/plain",
      ""
    );

    return;
  }

  http.end();

  Serial.println("PC SERVER NOT AVAILABLE");

  server.send(
    500,
    "text/html; charset=utf-8",
    "<html><meta charset='UTF-8'><body style='font-family:Arial;text-align:center;padding:50px'><h2>Không kết nối được PC Server</h2><p>Kiểm tra PC 192.168.2.23:8080 có đang chạy hay không.</p><a href='/'>← Quay lại Dashboard</a></body></html>"
  );
}

// ============================================================
// DASHBOARD
// ============================================================

void handleRoot()
{
  String page;

  page.reserve(18000);

  page += R"rawliteral(
<!DOCTYPE html>

<html>

<head>

<meta charset="UTF-8">

<meta name="viewport"
content="width=device-width,initial-scale=1">

<title>LuxMonitor V1.3</title>

<style>

*{
 box-sizing:border-box;
}

body{
 margin:0;
 background:#0e1116;
 color:#f2f4f7;
 font-family:Arial,Helvetica,sans-serif;
}

.header{
 padding:18px;
 background:#171b22;
 border-bottom:1px solid #292f38;
 position:sticky;
 top:0;
 z-index:10;
}

.read-settings{
 max-width:1200px;
 margin:14px auto 0;
 padding:12px 14px;
 background:#171c24;
 border:1px solid #292f38;
 border-radius:12px;
 display:flex;
 align-items:center;
 justify-content:space-between;
 gap:12px;
 flex-wrap:wrap;
}
.read-settings form{
 display:flex;
 align-items:center;
 gap:9px;
 flex-wrap:wrap;
 margin:0;
}
.read-settings label{
 color:#c4ccd6;
 font-size:13px;
}
.read-settings input{
 width:78px;
 padding:8px 9px;
 color:#fff;
 background:#252c36;
 border:1px solid #3a4350;
 border-radius:8px;
 font-size:14px;
}
.read-settings button{
 padding:9px 13px;
 color:#fff;
 background:#16885f;
 border:0;
 border-radius:8px;
 font-weight:bold;
 cursor:pointer;
}
.read-settings .hint{
 color:#8994a3;
 font-size:12px;
}
@media(max-width:600px){
 .read-settings{margin:10px 12px 0;}
 .read-settings form{width:100%;}
}

.header-inner{
 max-width:1200px;
 margin:auto;
 display:flex;
 justify-content:space-between;
 align-items:center;
 gap:15px;
}

.title{
 font-size:23px;
 font-weight:bold;
}

.sub{
 color:#8e99a8;
 font-size:13px;
 margin-top:4px;
}

.status{
 padding:8px 13px;
 border-radius:20px;
 font-size:13px;
 font-weight:bold;
 background:#303640;
}

.online{
 background:#123d2d;
 color:#55e0a1;
}

.offline{
 background:#402027;
 color:#ff7889;
}

.container{
 max-width:1200px;
 margin:auto;
 padding:18px;
}

.section-title{
 margin:25px 0 12px;
 color:#9ca7b5;
 font-size:14px;
 text-transform:uppercase;
 letter-spacing:1px;
}

/* ==========================================================
   UPDATE TIME - THEM MOI
   ========================================================== */

.power-title{
 display:flex;
 justify-content:space-between;
 align-items:center;
 gap:10px;
}

#lastupdate{
 color:#657080;
 font-size:12px;
 text-transform:none;
 letter-spacing:0;
 font-weight:normal;
 white-space:nowrap;
}

/* ==========================================================
   END UPDATE TIME
   ========================================================== */

.grid{
 display:grid;
 grid-template-columns:repeat(4,1fr);
 gap:12px;
}

.card{
 background:#181d25;
 border:1px solid #272d36;
 border-radius:16px;
 padding:17px;
 min-width:0;
}

.card-title{
 color:#9da8b7;
 font-size:13px;
 margin-bottom:9px;
}

.big{
 font-size:29px;
 font-weight:bold;
}

.unit{
 color:#8994a3;
 font-size:14px;
}

.small{
 color:#8994a3;
 font-size:12px;
 margin-top:9px;
 line-height:1.7;
}

.green{
 color:#4ee09e;
}

.blue{
 color:#64b8ff;
}

.orange{
 color:#ffb45c;
}

.red{
 color:#ff7080;
}

.yellow{
 color:#f0d66c;
}

.socbar{
 height:8px;
 background:#292f38;
 border-radius:10px;
 overflow:hidden;
 margin-top:12px;
}

.socfill{
 height:100%;
 width:0%;
 background:#4ee09e;
 transition:.4s;
}

.pvrow{
 display:flex;
 justify-content:space-between;
 padding:7px 0;
 border-bottom:1px solid #272d36;
}

.pvrow:last-child{
 border-bottom:0;
}

.energy{
 font-size:24px;
 font-weight:bold;
}

.temp{
 font-size:25px;
 font-weight:bold;
}

.footer{
 text-align:center;
 color:#657080;
 font-size:12px;
 padding:30px 10px;
}

.btn{
 display:inline-block;
 text-decoration:none;
 color:#fff;
 background:#252c36;
 padding:9px 13px;
 border-radius:9px;
 font-size:13px;
}

@media(max-width:900px){

 .grid{
   grid-template-columns:repeat(2,1fr);
 }

}

@media(max-width:550px){

 .header-inner{
   align-items:flex-start;
 }

 .title{
   font-size:19px;
 }

 .grid{
   grid-template-columns:1fr 1fr;
   gap:9px;
 }

 .card{
   padding:13px;
   border-radius:13px;
 }

 .big{
   font-size:23px;
 }

 .energy{
   font-size:20px;
 }

 /* UPDATE TIME MOBILE */

 .power-title{
   gap:5px;
 }

 #lastupdate{
   font-size:10px;
 }

}

</style>

</head>

<body>

<header class="header">

<div class="header-inner">

<div>

<div class="title">
⚡ LuxPower SNA 6K
</div>

<div class="sub">
LuxMonitor V1.3 · <span id="ip">---</span>
</div>

</div>

<div class="header-actions">

<a class="remote-btn" href="/remote">
🌐 GIÁM SÁT TỪ XA
</a>

<div id="status"
class="status offline">
OFFLINE
</div>

</div>

</div>

</header>

<div class="container">

<!-- POWER -->

<div class="section-title power-title">

<span>
Công suất hiện tại
</span>

<span id="lastupdate">
Cập nhật: --:--:--
</span>

</div>

<div class="grid">

<!-- PV -->

<div class="card">

<div class="card-title">
☀️ PV SOLAR
</div>

<div class="big green">
<span id="pvt">--</span>
<span class="unit">W</span>
</div>

<div class="small">

<div class="pvrow">
<span>PV1</span>
<span>
<span id="pv1w">--</span> W
/
<span id="pv1a">--</span> A
</span>
</div>

<div class="pvrow">
<span>PV2</span>
<span>
<span id="pv2w">--</span> W
/
<span id="pv2a">--</span> A
</span>
</div>

<div class="pvrow">
<span>Dòng PV tổng</span>
<span class="green">
<span id="pva">--</span> A
</span>
</div>

</div>

</div>

<!-- BATTERY -->

<div class="card">

<div class="card-title">
🔋 PIN
</div>

<div class="big">
<span id="soc">--</span>
<span class="unit">%</span>
</div>

<div class="socbar">
<div id="socfill" class="socfill"></div>
</div>

<div class="small">

Điện áp:
<span id="battv">--</span> V

<br>

SOH:
<span id="soh">--</span> %

<br>

Sạc:
<span class="green">
<span id="chargew">--</span> W /
<span id="chargea">--</span> A
</span>

<br>

Xả:
<span class="orange">
<span id="dischargew">--</span> W /
<span id="dischargea">--</span> A
</span>

</div>

</div>

<!-- GRID -->

<div class="card">

<div class="card-title">
⚡ LƯỚI
</div>

<div class="big blue">
<span id="gridw">--</span>
<span class="unit">W</span>
</div>

<div class="small">

Đẩy lưới:
<span id="exportw">--</span> W

<br>

Điện áp:
<span id="gridv">--</span> V

<br>

Tần số:
<span id="gridhz">--</span> Hz

</div>

</div>

<!-- LOAD -->

<div class="card">

<div class="card-title">
🏠 TẢI
</div>

<div class="big orange">
<span id="loadw">--</span>
<span class="unit">W</span>
</div>

<div class="small">

Biến tần:
<span id="invw">--</span> W

<br>

EPS:
<span id="epsw">--</span> W

</div>

</div>

</div>

<!-- DAILY -->

<div class="section-title">
  Năng lượng hôm nay
</div>

<div class="grid daily-grid">

  <!-- PV -->
  <div class="card">
    <div class="card-title">☀️ PV</div>
    <div class="energy green">
      <span id="pvday">--</span>
      <span class="unit">kWh</span>
    </div>
  </div>

  <!-- TIÊU THỤ -->
  <div class="card">
    <div class="card-title">🏠 TIÊU THỤ</div>
    <div class="energy">
      <span id="consumptionday">--</span>
      <span class="unit">kWh</span>
    </div>
  </div>

  <!-- SẠC -->
  <div class="card">
    <div class="card-title">🔋 SẠC</div>
    <div class="energy">
      <span id="chargeday">--</span>
      <span class="unit">kWh</span>
    </div>
  </div>

  <!-- XẢ -->
  <div class="card">
    <div class="card-title">🔋 XẢ</div>
    <div class="energy orange">
      <span id="dischargeday">--</span>
      <span class="unit">kWh</span>
    </div>
  </div>

  <!-- LẤY LƯỚI -->
  <div class="card">
    <div class="card-title">⚡ LẤY LƯỚI</div>
    <div class="energy blue">
      <span id="griday">--</span>
      <span class="unit">kWh</span>
    </div>
  </div>

  <!-- ĐẨY LƯỚI -->
  <div class="card">
    <div class="card-title">⚡ ĐẨY LƯỚI</div>
    <div class="energy">
      <span id="exportday">--</span>
      <span class="unit">kWh</span>
    </div>
  </div>

</div>

<!-- LIFETIME -->

<div class="section-title">
Tổng năng lượng tích lũy
</div>

<div class="grid">

<div class="card">
<div class="card-title">☀️ PV</div>
<div class="energy green">
<span id="pvall">--</span>
<span class="unit">kWh</span>
</div>
</div>

<div class="card">
<div class="card-title">🏠 TIÊU THỤ</div>
<div class="energy">
<span id="consumptionall">--</span>
<span class="unit">kWh</span>
</div>
</div>

<div class="card">
<div class="card-title">🔋 SẠC</div>
<div class="energy">
<span id="chargeall">--</span>
<span class="unit">kWh</span>
</div>
</div>

<div class="card">
<div class="card-title">🔋 XẢ</div>
<div class="energy orange">
<span id="dischargeall">--</span>
<span class="unit">kWh</span>
</div>
</div>

<div class="card">
<div class="card-title">⚡ LẤY LƯỚI</div>
<div class="energy blue">
<span id="gridimportall">--</span>
<span class="unit">kWh</span>
</div>
</div>

<div class="card">
<div class="card-title">⚡ ĐẨY LƯỚI</div>
<div class="energy">
<span id="exportall">--</span>
<span class="unit">kWh</span>
</div>
</div>


</div>

<!-- TEMPERATURE -->

<div class="section-title">
Nhiệt độ
</div>

<div class="grid">

<div class="card">

<div class="card-title">
BIẾN TẦN
</div>

<div class="temp">
<span id="invt">--</span>
<span class="unit">°C</span>
</div>

</div>

<div class="card">

<div class="card-title">
AC
</div>

<div class="temp">
<span id="act">--</span>
<span class="unit">°C</span>
</div>

</div>

<div class="card">

<div class="card-title">
DC
</div>

<div class="temp">
<span id="dct">--</span>
<span class="unit">°C</span>
</div>

</div>

<div class="card">

<div class="card-title">
PIN
</div>

<div class="temp">
<span id="battt">--</span>
<span class="unit">°C</span>
</div>

</div>

</div>

<!-- DIAGNOSTIC -->

<div class="section-title">
Thông tin hệ thống
</div>

<div class="grid">


<div class="card">

<div class="card-title">
THỜI GIAN HOẠT ĐỘNG
</div>

<div class="small"
style="font-size:16px;color:white">

<span id="runtime">--</span>

</div>

</div>

<div class="card">

<div class="card-title">
WIFI/BIẾN TẦN
</div>

<div style="display:flex;gap:8px;flex-wrap:wrap">

<a class="btn" href="/wifi">
⚙ Cài đặt WiFi
</a>

<a class="btn" href="/inverter">
⚙ Cài đặt biến tần
</a>

</div>

</div>

<div class="read-settings">
  <form action="/savereadinterval" method="POST">
    <label for="readSeconds">Chu kỳ đọc dữ liệu LuxPower:</label>
    <input id="readSeconds" name="seconds" type="number"
      min="2" max="30" step="1" required value=")rawliteral";

  page += String(READ_INTERVAL / 1000UL);

  page += R"rawliteral(">
    <span class="hint">giây (từ 2 đến 30, chỉnh từng giây)</span>
    <button type="submit">Áp dụng</button>
  </form>
</div>

</div>

</div>

<div class="footer">

LuxMonitor V1.3<br>

READ ONLY · Auto update <br>

<span id="update">
Đang chờ dữ liệu...
</span>

</div>

<script>

function set(id,value)
{
  let e=document.getElementById(id);

  if(e)
    e.textContent=value;
}

function updateData(d)
{

  // STATUS

  let status=document.getElementById("status");

  if(d.online)
  {
    status.textContent="ONLINE";
    status.className="status online";
  }
  else
  {
    status.textContent="OFFLINE";
    status.className="status offline";
  }

  // IP

  set("ip",d.ip);

  // ==========================================================
  // THOI GIAN CAP NHAT - THEM MOI
  // ==========================================================

  set(
  "lastupdate",
  "Cập nhật: " + d.updatetime
);

  // ==========================================================
  // PV
  // ==========================================================

  set("pvt",Math.round(d.pvt));

  set("pv1w",Math.round(d.pv1w));
  set("pv1a",d.pv1a.toFixed(1));

  set("pv2w",Math.round(d.pv2w));
  set("pv2a",d.pv2a.toFixed(1));

  set("pva",d.pva.toFixed(1));

  // BATTERY

  set("soc",d.soc);

  set("battv",d.battv.toFixed(1));

  set("soh",d.soh);

  set("chargew",Math.round(d.chargew));
  set("chargea",d.chargea.toFixed(1));

  set("dischargew",Math.round(d.dischargew));
  set("dischargea",d.dischargea.toFixed(1));

  let socfill=document.getElementById("socfill");

  socfill.style.width=
    Math.max(
      0,
      Math.min(100,d.soc)
    )+"%";

  // GRID

  set("gridw",Math.round(d.gridw));

  set("exportw",Math.round(d.exportw));

  set("gridv",d.gridv.toFixed(1));

  set("gridhz",d.gridhz.toFixed(2));

  // LOAD

  set("loadw",Math.round(d.loadw));

  set("invw",Math.round(d.invw));

  set("epsw",Math.round(d.epsw));

  // DAILY

  set("pvday",d.pvday.toFixed(1));
  set("chargeday",d.chargeday.toFixed(1));
  set("dischargeday",d.dischargeday.toFixed(1));
  set("griday",d.griday.toFixed(1));
  set("exportday",d.exportday.toFixed(1));
  set("consumptionday",d.consumptionday.toFixed(1));

  // LIFETIME

  // LIFETIME

set("pvall",d.pvall.toFixed(1));

set("consumptionall",d.consumptionall.toFixed(1));

set("chargeall",d.chargeall.toFixed(1));

set("dischargeall",d.dischargeall.toFixed(1));

set("exportall",d.exportall.toFixed(1));

set("gridimportall",d.gridimportall.toFixed(1));

  // TEMPERATURE

  set("invt",d.invt);
  set("act",d.act);
  set("dct",d.dct);
  set("battt",d.battt);

  // DIAGNOSTIC

  set("fault",d.fault);
  set("warning",d.warning);
  set("runtime",d.runtime);

  // UPDATE

  if(d.age < 1000)
  {
    set(
      "update",
      "Cập nhật vừa xong"
    );
  }
  else
  {
    set(
      "update",
      "Cập nhật " +
      Math.floor(d.age/1000) +
      " giây trước"
    );
  }

}

let requestRunning = false;

async function getData()
{
  // Không cho request mới chạy khi request trước
  // vẫn chưa hoàn thành.
  if(requestRunning)
    return;

  requestRunning = true;

  try
  {
    let response =
      await fetch(
        "/api?t=" + Date.now(),
        {
          cache:"no-store"
        }
      );

    if(!response.ok)
      throw new Error("HTTP " + response.status);

    let data =
      await response.json();

    updateData(data);
  }
  catch(error)
  {
    let status =
      document.getElementById("status");

    status.textContent="MẤT KẾT NỐI";

    status.className="status offline";
  }
  finally
  {
    requestRunning = false;
  }
}

getData();

setInterval(
  getData,
  2000
);

</script>

</body>

</html>
)rawliteral";

  server.send(
    200,
    "text/html; charset=utf-8",
    page
  );
}

// ============================================================
// SETUP
// ============================================================

void setup()
{
  Serial.begin(115200);

  delay(500);

  Serial.println();
  Serial.println();
  Serial.println("================================================");
  Serial.println("          LUXMONITOR V1.3");
  Serial.println("          LUXPOWER SNA 6K");
  Serial.println("================================================");

  Serial.print("Dongle   : ");
Serial.println(dongleSN);

Serial.print("Inverter : ");
Serial.println(inverterSN);

Serial.print("IP       : ");
Serial.println(inverterIP);

Serial.print("Port     : ");
Serial.println(inverterPort);

  // ----------------------------------------------------------
// INVERTER CONFIG
// ----------------------------------------------------------

loadInverter();
loadReadInterval();

Serial.print("READ INTERVAL: ");
Serial.print(READ_INTERVAL / 1000UL);
Serial.println(" seconds");

Serial.println();
Serial.println("=================================");
Serial.println(" INVERTER CONFIG");
Serial.println("=================================");

Serial.print("IP       : ");
Serial.println(inverterIP);

Serial.print("Port     : ");
Serial.println(inverterPort);

Serial.print("Dongle   : ");
Serial.println(dongleSN);

Serial.print("Inverter : ");
Serial.println(inverterSN);

// ----------------------------------------------------------
// WIFI
// ----------------------------------------------------------

connectWiFi();

startRemote();

  // ----------------------------------------------------------
  // OLED UDP
  // ----------------------------------------------------------

  if(WiFi.status() == WL_CONNECTED)
  {
    oledUDP.begin(OLED_LOCAL_PORT);

    Serial.println();
    Serial.println("OLED UDP STARTED");
    Serial.print("Broadcast: ");
    Serial.print("192.168.2.255:");
    Serial.println(OLED_UDP_PORT);
  }

  // ----------------------------------------------------------
  // WEB SERVER
  // ----------------------------------------------------------

  server.on(
    "/",
    HTTP_GET,
    handleRoot
  );

    server.on(
    "/remote",
    HTTP_GET,
    handleRemote
  );

  server.on(
  "/inverter",
  HTTP_GET,
  handleInverter
);

  server.on(
    "/api",
    HTTP_GET,
    handleAPI
  );

  server.on(
    "/savereadinterval",
    HTTP_POST,
    handleSaveReadInterval
  );

  server.on(
    "/wifi",
    HTTP_GET,
    handleWiFi
  );

  server.on(
    "/savewifi",
    HTTP_POST,
    handleSaveWiFi
  );

  server.on(
  "/saveinverter",
  HTTP_POST,
  handleSaveInverter
);

  server.on(
    "/clearwifi",
    HTTP_GET,
    handleClearWiFi
  );

  server.begin();

  Serial.println();
  Serial.println("Web server started");

  if (WiFi.getMode() == WIFI_AP)
  {
    Serial.print("Dashboard: http://");
    Serial.print(WiFi.softAPIP());
    Serial.println("/");
  }
  else
  {
    Serial.print("Dashboard: http://");
    Serial.print(WiFi.localIP());
    Serial.println("/");
  }

  Serial.println("================================================");

  // ----------------------------------------------------------
  // CLEAR DATA
  // ----------------------------------------------------------

  for (int i = 0; i < 172; i++)
    regData[i] = 0;
}

// ============================================================
// LOOP
// ============================================================

void loop()
{
  // ==========================================================
  // REMOTE WEBSOCKET
  // Chạy liên tục, ưu tiên xử lý kết nối
  // ==========================================================

  if(remoteEnable)
  {
    remoteWS.loop();
    sendRemoteData();
  }

  // ==========================================================
  // LOCAL WEB SERVER
  // ==========================================================

  server.handleClient();

  // ==========================================================
  // READ MAIN DATA - REG0 TO REG70
  // GIỮ NGUYÊN ENGINE ĐANG HOẠT ĐỘNG
  // ==========================================================

  if(
    millis() - lastReadAttempt >= READ_INTERVAL
  )
  {
    lastReadAttempt = millis();

    bool ok = readLuxPower();

    if(!ok)
    {
      if(
        millis() - lastReadOK >
        DISCONNECT_TIME
      )
      {
        luxConnected = false;
      }
    }
  }

  // ==========================================================
  // REMOTE WEBSOCKET - XỬ LÝ LẠI SAU KHI ĐỌC LUXPOWER
  // ==========================================================

  if(remoteEnable)
  {
    remoteWS.loop();
    sendRemoteData();
  }

   // ==========================================================
  // OLED ESP32
  // ==========================================================

  sendOLEDData();

  yield();
}
