const WebSocket = require('ws');
const express = require('express');
const fs = require('fs');
const path = require('path');
const app = express();
const server = require('http').createServer(app);
const wss = new WebSocket.Server({ server });

// File path để lưu device names
const DEVICE_NAMES_FILE = path.join(__dirname, 'device-names.json');

// Store connected ESP32 devices and web clients separately
let connectedDevices = new Map();
let webClients = new Set();
let deviceNames = new Map();

// Load saved device names from file
function loadDeviceNames() {
    try {
        if (fs.existsSync(DEVICE_NAMES_FILE)) {
            const data = fs.readFileSync(DEVICE_NAMES_FILE, 'utf8');
            const names = JSON.parse(data);
            deviceNames = new Map(Object.entries(names));
            console.log('Loaded device names:', deviceNames);
        }
    } catch (error) {
        console.error('Error loading device names:', error);
    }
}

// Save device names to file
function saveDeviceNames() {
    try {
        const names = Object.fromEntries(deviceNames);
        fs.writeFileSync(DEVICE_NAMES_FILE, JSON.stringify(names, null, 2));
        console.log('Saved device names:', names);
    } catch (error) {
        console.error('Error saving device names:', error);
    }
}

// Load saved names at startup
loadDeviceNames();

wss.on('connection', (ws) => {
    console.log('New connection received');
    let deviceId = null;
    let isWebClient = false;

    ws.on('message', (message) => {
        try {
            const data = JSON.parse(message);
            
            if (data.type === 'register') {
                // Đây là ESP32 device
                deviceId = data.deviceId;
                // Sử dụng tên đã lưu hoặc tên mặc định
                const savedName = deviceNames.get(deviceId) || `ESP32-${deviceId}`;
                
                connectedDevices.set(deviceId, {
                    name: savedName,
                    lightStatus: false,
                    ws: ws
                });
                console.log(`ESP32 device connected - ID: ${deviceId}, Name: ${savedName}`);
                
                broadcastToWeb({
                    type: 'notification',
                    message: `Device "${savedName}" connected`
                });
                
                broadcastDeviceList();
            } else if (data.type === 'webClient') {
                isWebClient = true;
                webClients.add(ws);
                console.log('Web client connected');
                sendDeviceList(ws);
            } else if (data.type === 'updateName') {
                if (connectedDevices.has(data.deviceId)) {
                    const device = connectedDevices.get(data.deviceId);
                    const oldName = device.name;
                    device.name = data.name;
                    
                    // Lưu tên mới vào Map và file
                    deviceNames.set(data.deviceId, data.name);
                    saveDeviceNames();
                    
                    console.log(`Device ${data.deviceId} renamed from "${oldName}" to "${data.name}"`);
                    
                    broadcastToWeb({
                        type: 'notification',
                        message: `Device "${oldName}" renamed to "${data.name}"`
                    });
                    
                    broadcastDeviceList();
                }
            } else if (data.type === 'toggleLight') {
                if (connectedDevices.has(data.deviceId)) {
                    const device = connectedDevices.get(data.deviceId);
                    device.lightStatus = data.status;
                    device.ws.send(JSON.stringify({
                        type: 'lightCommand',
                        status: data.status
                    }));
                    
                    broadcastToWeb({
                        type: 'notification',
                        message: `Light ${data.status ? 'turned ON' : 'turned OFF'} for device "${device.name}"`
                    });
                    
                    broadcastDeviceList();
                }
            }
        } catch (error) {
            console.error('Error processing message:', error);
        }
    });

    ws.on('close', () => {
        if (isWebClient) {
            webClients.delete(ws);
            console.log('Web client disconnected');
        } else if (deviceId && connectedDevices.has(deviceId)) {
            const deviceName = connectedDevices.get(deviceId).name;
            connectedDevices.delete(deviceId);
            console.log(`ESP32 device disconnected - ID: ${deviceId}, Name: ${deviceName}`);
            
            broadcastToWeb({
                type: 'notification',
                message: `Device "${deviceName}" disconnected`
            });
            
            broadcastDeviceList();
        }
    });
});

function sendDeviceList(ws) {
    const deviceList = Array.from(connectedDevices.entries()).map(([id, device]) => ({
        id: id,
        name: device.name,
        lightStatus: device.lightStatus
    }));

    ws.send(JSON.stringify({
        type: 'deviceList',
        devices: deviceList
    }));
}

function broadcastDeviceList() {
    const deviceList = Array.from(connectedDevices.entries()).map(([id, device]) => ({
        id: id,
        name: device.name,
        lightStatus: device.lightStatus
    }));

    console.log('Broadcasting device list:', deviceList); // Kiểm tra dữ liệu danh sách thiết bị
    broadcastToWeb({
        type: 'deviceList',
        devices: deviceList
    });
}


function broadcastToWeb(message) {
    const messageStr = JSON.stringify(message);
    webClients.forEach((client) => {
        if (client.readyState === WebSocket.OPEN) {
            client.send(messageStr);
        }
    });
}

app.use(express.static('public'));
server.listen(3000, () => {
    console.log('Server running on port 3000');
});