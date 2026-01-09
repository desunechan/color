// main_app.cpp - Complete unified application
#include "tray_icon.h"
#include "hotkey_manager.h"
#include "process_utils.h"
#include "colorant.h"
#include <thread>
#include <atomic>
#include <mutex>
#include <cmath>

class ColorantApp {
private:
    TrayIcon trayIcon;
    HotkeyManager hotkeyManager;
    Colorant* colorantInstance;
    std::mutex instanceMutex;
    
    std::atomic<bool> appRunning;
    std::atomic<bool> isRecording;
    
    int screenX, screenY, xfov, yfov;
    float flickspeed, movespeed;
    
    void InitializeColorant();
    void CleanupColorantInstance();  // RENAMED
    
public:
    ColorantApp();
    ~ColorantApp();
    
    bool Initialize(HINSTANCE hInstance);
    void Run();
    void Stop();
    
    void OnToggleRecording();
    void OnExit();
};

ColorantApp::ColorantApp() 
    : colorantInstance(nullptr), 
      appRunning(true), 
      isRecording(false),
      screenX(0), screenY(0), xfov(75), yfov(75),
      flickspeed(1.0f), movespeed(1.0f) {
    
    // Get screen center
    screenX = GetSystemMetrics(SM_CXSCREEN) / 2 - xfov / 2;
    screenY = GetSystemMetrics(SM_CYSCREEN) / 2 - yfov / 2;
    
    // Calculate speeds (from Python formula)
    float ingame_sensitivity = 0.23f;
    flickspeed = 1.07437623f * powf(ingame_sensitivity, -0.9936827126f);
    movespeed = 1.0f / (10.0f * ingame_sensitivity);
}

ColorantApp::~ColorantApp() {
    Stop();
}

bool ColorantApp::Initialize(HINSTANCE hInstance) {
    // Hide console
    ProcessUtils::HideConsole();
    
    // Spoof process name
    ProcessUtils::SpoofProcessName();
    
    // Create tray icon
    if (!trayIcon.Create(hInstance)) {
        return false;
    }
    
    // Setup tray callbacks
    trayIcon.onToggle = [this]() { OnToggleRecording(); };
    trayIcon.onExit = [this]() { OnExit(); };
    
    trayIcon.Show();
    trayIcon.UpdateIcon(false);
    
    // Setup hotkeys
    hotkeyManager.RegisterHotkey(VK_F1, [this]() { OnToggleRecording(); });
    hotkeyManager.Start();
    
    return true;
}

void ColorantApp::InitializeColorant() {
    std::lock_guard<std::mutex> lock(instanceMutex);
    if (!colorantInstance) {
        colorantInstance = ::CreateColorant(  // Use global function
            screenX, screenY, xfov, yfov, 
            flickspeed, movespeed
        );
    }
}

void ColorantApp::CleanupColorantInstance() {  // RENAMED
    std::lock_guard<std::mutex> lock(instanceMutex);
    if (colorantInstance) {
        ::CleanupColorant(colorantInstance);  // Use global function
        colorantInstance = nullptr;
    }
}

void ColorantApp::OnToggleRecording() {
    std::lock_guard<std::mutex> lock(instanceMutex);
    
    if (!colorantInstance) {
        colorantInstance = ::CreateColorant(  // Use global function
            screenX, screenY, xfov, yfov, 
            flickspeed, movespeed
        );
    }
    
    if (colorantInstance) {
        isRecording = ::ToggleColorant(colorantInstance);  // Use global function
        trayIcon.UpdateIcon(isRecording);
    }
}

void ColorantApp::OnExit() {
    Stop();
}

void ColorantApp::Run() {
    MSG msg;
    
    while (appRunning) {
        // Process Windows messages
        while (PeekMessage(&msg, nullptr, 0, 0, PM_REMOVE)) {
            if (msg.message == WM_QUIT) {
                appRunning = false;
                break;
            }
            TranslateMessage(&msg);
            DispatchMessage(&msg);
        }
        
        // Process tray icon messages
        trayIcon.ProcessMessages();
        
        // Small sleep to prevent CPU hogging
        Sleep(50);
    }
}

void ColorantApp::Stop() {
    appRunning = false;
    hotkeyManager.Stop();
    CleanupColorantInstance();  // Changed to new name
    trayIcon.Hide();
}

// Entry point
int WINAPI WinMain(HINSTANCE hInstance, HINSTANCE hPrevInstance, 
                   LPSTR lpCmdLine, int nCmdShow) {
    
    ColorantApp app;
    
    if (!app.Initialize(hInstance)) {
        MessageBoxA(nullptr, "Failed to initialize application", 
                   "Error", MB_ICONERROR);
        return 1;
    }
    
    app.Run();
    
    return 0;
}

//main_app.h
#pragma once
#include "tray_icon.h"
#include "hotkey_manager.h"
#include "process_utils.h"
#include "colorant.h"
#include <atomic>
#include <mutex>

class ColorantApp {
private:
    TrayIcon trayIcon;
    HotkeyManager hotkeyManager;
    Colorant* colorantInstance;
    std::mutex instanceMutex;
    
    std::atomic<bool> appRunning;
    std::atomic<bool> isRecording;
    
    int screenX, screenY, xfov, yfov;
    float flickspeed, movespeed;
    
    void InitializeColorant();
    void CleanupColorantInstance();  // Rename this
    
public:
    ColorantApp();
    ~ColorantApp();
    
    bool Initialize(HINSTANCE hInstance);
    void Run();
    void Stop();
    
    void OnToggleRecording();
    void OnExit();
};

//tray_icon.h
#include "tray_icon.h"
#include <thread>
#include <chrono>

TrayIcon* g_trayInstance = nullptr;

LRESULT CALLBACK TrayIcon::WindowProc(HWND hwnd, UINT msg, WPARAM wParam, LPARAM lParam) {
    if (g_trayInstance && g_trayInstance->hwnd == hwnd) {
        switch (msg) {
            case WM_USER + 1: {
                switch (lParam) {
                    case WM_LBUTTONDBLCLK:
                    case WM_LBUTTONUP:
                        if (g_trayInstance->onToggle) {
                            g_trayInstance->onToggle();
                        }
                        break;
                        
                    case WM_RBUTTONUP: {
                        POINT pt;
                        GetCursorPos(&pt);
                        
                        HMENU hMenu = CreatePopupMenu();
                        AppendMenuA(hMenu, MF_STRING, 1001, "Start/Stop Record (F1)");
                        AppendMenuA(hMenu, MF_SEPARATOR, 0, NULL);
                        AppendMenuA(hMenu, MF_STRING, 1002, "Exit");
                        
                        SetForegroundWindow(hwnd);
                        UINT clicked = TrackPopupMenu(hMenu, 
                            TPM_RETURNCMD | TPM_NONOTIFY, 
                            pt.x, pt.y, 0, hwnd, NULL);
                        DestroyMenu(hMenu);
                        
                        if (clicked == 1001 && g_trayInstance->onToggle) {
                            g_trayInstance->onToggle();
                        } else if (clicked == 1002 && g_trayInstance->onExit) {
                            g_trayInstance->onExit();
                        }
                        break;
                    }
                }
                break;
            }
            
            case WM_COMMAND:
                if (LOWORD(wParam) == 1001 && g_trayInstance->onToggle) {
                    g_trayInstance->onToggle();
                } else if (LOWORD(wParam) == 1002 && g_trayInstance->onExit) {
                    g_trayInstance->onExit();
                }
                break;
                
            case WM_DESTROY:
                PostQuitMessage(0);
                break;
        }
        
        if (msg == g_trayInstance->wm_taskbarcreated) {
            g_trayInstance->Show();
        }
    }
    
    return DefWindowProc(hwnd, msg, wParam, lParam);
}

TrayIcon::TrayIcon() : hwnd(nullptr), visible(false) {
    memset(&nid, 0, sizeof(nid));
    g_trayInstance = this;
}

TrayIcon::~TrayIcon() {
    Hide();
    if (nid.hIcon && nid.hIcon != (HICON)1) {
        DestroyIcon(nid.hIcon);
    }
    if (hwnd) {
        DestroyWindow(hwnd);
    }
    g_trayInstance = nullptr;
}

bool TrayIcon::Create(HINSTANCE hInstance) {
    wm_taskbarcreated = RegisterWindowMessageA("TaskbarCreated");
    
    WNDCLASSEXA wc = {0};
    wc.cbSize = sizeof(WNDCLASSEXA);
    wc.lpfnWndProc = WindowProc;
    wc.hInstance = hInstance;
    wc.lpszClassName = "TrayIconWindowClass";
    
    if (!RegisterClassExA(&wc)) {
        return false;
    }
    
    hwnd = CreateWindowExA(0, wc.lpszClassName, "TrayIcon", 0,
        0, 0, 0, 0, nullptr, nullptr, hInstance, nullptr);
    
    if (!hwnd) {
        return false;
    }
    
    nid.cbSize = sizeof(NOTIFYICONDATA);
    nid.hWnd = hwnd;
    nid.uID = 1;
    nid.uFlags = NIF_ICON | NIF_MESSAGE | NIF_TIP;
    nid.uCallbackMessage = WM_USER + 1;
    
    // Create default icon
    HICON hIcon = (HICON)LoadImage(nullptr, IDI_APPLICATION, 
        IMAGE_ICON, 16, 16, LR_SHARED);
    nid.hIcon = hIcon;
    
    strcpy_s(nid.szTip, "Colorant Recorder");
    
    return true;
}

void TrayIcon::Show() {
    if (!visible) {
        Shell_NotifyIconA(NIM_ADD, &nid);
        visible = true;
    }
}

void TrayIcon::Hide() {
    if (visible) {
        Shell_NotifyIconA(NIM_DELETE, &nid);
        visible = false;
    }
}

void TrayIcon::UpdateIcon(bool recording) {
    if (!hwnd) return;
    
    // Create colored icon
    HDC hdc = GetDC(nullptr);
    HDC hdcMem = CreateCompatibleDC(hdc);
    HBITMAP hBitmap = CreateCompatibleBitmap(hdc, 16, 16);
    SelectObject(hdcMem, hBitmap);
    
    HBRUSH hBrush;
    if (recording) {
        hBrush = CreateSolidBrush(RGB(255, 0, 0));
    } else {
        hBrush = CreateSolidBrush(RGB(100, 100, 100));
    }
    
    RECT rect = {0, 0, 16, 16};
    FillRect(hdcMem, &rect, hBrush);
    
    // Draw circle outline
    HPEN hPen = CreatePen(PS_SOLID, 1, recording ? RGB(255, 255, 255) : RGB(50, 50, 50));
    SelectObject(hdcMem, hPen);
    SelectObject(hdcMem, GetStockObject(NULL_BRUSH));
    Ellipse(hdcMem, 2, 2, 14, 14);
    
    if (recording) {
        SetTextColor(hdcMem, RGB(255, 255, 255));
        SetBkMode(hdcMem, TRANSPARENT);
        DrawTextA(hdcMem, "R", 1, &rect, DT_CENTER | DT_VCENTER | DT_SINGLELINE);
    }
    
    // Create icon from bitmap
    ICONINFO iconInfo;
    iconInfo.fIcon = TRUE;
    iconInfo.hbmMask = hBitmap;
    iconInfo.hbmColor = hBitmap;
    
    HICON hIcon = CreateIconIndirect(&iconInfo);
    
    if (hIcon) {
        if (nid.hIcon && nid.hIcon != (HICON)1) {
            DestroyIcon(nid.hIcon);
        }
        nid.hIcon = hIcon;
        
        // Update tooltip
        if (recording) {
            strcpy_s(nid.szTip, "Recording - Press F1 to stop");
        } else {
            strcpy_s(nid.szTip, "Press F1 to record");
        }
        
        Shell_NotifyIconA(NIM_MODIFY, &nid);
    }
    
    // Cleanup GDI objects
    DeleteObject(hBrush);
    DeleteObject(hPen);
    DeleteObject(hBitmap);
    DeleteDC(hdcMem);
    ReleaseDC(nullptr, hdc);
}

void TrayIcon::SetTooltip(const std::string& text) {
    strncpy_s(nid.szTip, text.c_str(), 127);
    if (visible) {
        Shell_NotifyIconA(NIM_MODIFY, &nid);
    }
}

void TrayIcon::ProcessMessages() {
    MSG msg;
    while (PeekMessage(&msg, hwnd, 0, 0, PM_REMOVE)) {
        TranslateMessage(&msg);
        DispatchMessage(&msg);
    }
}

//colorant.cpp
#include "colorant.h"
#include "capture.h"
#include "mouse.h"
#include <chrono>
#include <algorithm>

Colorant::Colorant(int x, int y, int xfov, int yfov, float FLICKSPEED, float MOVESPEED)
    : flickspeed(FLICKSPEED), movespeed(MOVESPEED),
    toggled(false), window_toggled(false), running(true) {

    LOWER_COLOR = cv::Scalar(140, 120, 180);
    UPPER_COLOR = cv::Scalar(160, 200, 255);

    mouse = new ArduinoMouse();
    grabber = new Capture(x, y, xfov, yfov);

    listen_thread = std::thread(&Colorant::ListenThread, this);
}

Colorant::~Colorant() {
    Close();
}

void Colorant::ListenThread() {
    while (running) {
        if (GetAsyncKeyState(VK_F2) & 0x8000) {
            window_toggled = !window_toggled;
            Sleep(200);
        }

        if (toggled) {
            if ((GetAsyncKeyState(VK_LSHIFT) & 0x8000) ||
                (GetAsyncKeyState(VK_LBUTTON) & 0x8000) ||
                (GetAsyncKeyState(VK_LCONTROL) & 0x8000) ||
                (GetAsyncKeyState('F') & 0x8000) ||
                (GetAsyncKeyState(VK_SPACE) & 0x8000)) {
                ProcessAction("move");
            }
            else if (GetAsyncKeyState(VK_MENU) & 0x8000) {
                ProcessAction("click");
            }
            else if (GetAsyncKeyState(VK_F12) & 0x8000) {
                ProcessAction("flick");
            }
        }

        Sleep(10);
    }
}

void Colorant::ProcessAction(const std::string& action) {
    if (!toggled) return;

    cv::Mat screen = grabber->GetScreen();
    if (screen.empty()) return;

    cv::Mat hsv;
    cv::cvtColor(screen, hsv, cv::COLOR_BGR2HSV);

    cv::Mat mask;
    cv::inRange(hsv, LOWER_COLOR, UPPER_COLOR, mask);

    cv::Mat dilated;
    cv::dilate(mask, dilated, cv::Mat(), cv::Point(-1, -1), 5);

    std::vector<std::vector<cv::Point>> contours;
    cv::findContours(dilated, contours, cv::RETR_EXTERNAL, cv::CHAIN_APPROX_SIMPLE);

    if (contours.empty()) return;

    auto contour = *std::max_element(contours.begin(), contours.end(),
        [](const std::vector<cv::Point>& a, const std::vector<cv::Point>& b) {
            return cv::contourArea(a) < cv::contourArea(b);
        });

    cv::Rect rect = cv::boundingRect(contour);
    int x = rect.x;
    int y = rect.y;
    int w = rect.width;
    int h = rect.height;

    cv::Point center(x + w / 2, y + h / 2);
    int y_offset = static_cast<int>(h * 0.1);

    if (action == "move") {
        int cX = center.x;
        int cY = y + y_offset;
        int x_diff = cX - grabber->GetXFov() / 2;
        int y_diff = cY - grabber->GetYFov() / 2;
        mouse->Move(x_diff * movespeed, y_diff * movespeed);
    }
    else if (action == "click") {
        if (abs(center.x - grabber->GetXFov() / 2) <= 4 &&
            abs(center.y - grabber->GetYFov() / 2) <= 10) {
            mouse->Click();
        }
    }
    else if (action == "flick") {
        int cX = center.x + 2;
        int cY = y + y_offset;
        int x_diff = cX - grabber->GetXFov() / 2;
        int y_diff = cY - grabber->GetYFov() / 2;

        float flickx = x_diff * flickspeed;
        float flicky = y_diff * flickspeed;

        mouse->Flick(flickx, flicky);
        mouse->Click();
        mouse->Flick(-flickx, -flicky);
    }
}

bool Colorant::Toggle() {
    toggled = !toggled;
    return toggled;
}

bool Colorant::IsToggled() const {
    return toggled;
}

void Colorant::Close() {
    running = false;

    if (grabber) {
        grabber->Stop();
        delete grabber;
        grabber = nullptr;
    }

    if (mouse) {
        mouse->Close();
        delete mouse;
        mouse = nullptr;
    }

    if (listen_thread.joinable()) {
        listen_thread.join();
    }
}

// Simple factory functions (not DLL exports)
Colorant* CreateColorant(int x, int y, int xfov, int yfov, float FLICKSPEED, float MOVESPEED) {
    return new Colorant(x, y, xfov, yfov, FLICKSPEED, MOVESPEED);
}

bool ToggleColorant(Colorant* instance) {
    if (instance) {
        return instance->Toggle();
    }
    return false;
}

void CleanupColorant(Colorant* instance) {
    if (instance) {
        delete instance;
    }
}

// process_utils.h
#pragma once
#include <windows.h>
#include <string>

class ProcessUtils {
public:
    static bool SpoofProcessName();
    static bool HideConsole();
    static bool SetWindowTitle(const std::string& title);
};

// process_utils.cpp
#include "process_utils.h"
#include <TlHelp32.h>
#include <iostream>

bool ProcessUtils::SpoofProcessName() {
    // Method 1: Set window title
    SetWindowTitle("svchost.exe -k LocalSystemNetworkRestricted");
    
    // Method 2: Try to modify process name in memory (advanced)
    // Note: This is simplified - actual spoofing requires more complex techniques
    return true;
}

bool ProcessUtils::HideConsole() {
    HWND console = GetConsoleWindow();
    if (console) {
        ShowWindow(console, SW_HIDE);
        return true;
    }
    return false;
}

bool ProcessUtils::SetWindowTitle(const std::string& title) {
    return SetConsoleTitleA(title.c_str()) != 0;
}

// hotkey_manager.cpp
#include "hotkey_manager.h"
#include <chrono>

HotkeyManager* g_hotkeyManager = nullptr;

LRESULT CALLBACK HotkeyManager::KeyboardHook(int nCode, WPARAM wParam, LPARAM lParam) {
    if (nCode >= 0 && wParam == WM_KEYDOWN) {
        KBDLLHOOKSTRUCT* pKey = (KBDLLHOOKSTRUCT*)lParam;
        
        if (g_hotkeyManager) {
            auto it = g_hotkeyManager->hotkeys.find(pKey->vkCode);
            if (it != g_hotkeyManager->hotkeys.end()) {
                DWORD currentTime = GetTickCount();
                if (currentTime - it->second.lastTrigger > g_hotkeyManager->debounceTime) {
                    it->second.lastTrigger = currentTime;
                    if (it->second.callback) {
                        it->second.callback();
                    }
                }
            }
        }
    }
    
    return CallNextHookEx(nullptr, nCode, wParam, lParam);
}

HotkeyManager::HotkeyManager(DWORD debounceMs) 
    : debounceTime(debounceMs), running(false), keyboardHook(nullptr) {
    g_hotkeyManager = this;
}

HotkeyManager::~HotkeyManager() {
    Stop();
    g_hotkeyManager = nullptr;
}

bool HotkeyManager::RegisterHotkey(UINT vkCode, std::function<void()> callback) {
    HotkeyInfo info;
    info.vkCode = vkCode;
    info.callback = callback;
    info.lastTrigger = 0;
    
    hotkeys[vkCode] = info;
    return true;
}

bool HotkeyManager::UnregisterHotkey(UINT vkCode) {
    return hotkeys.erase(vkCode) > 0;
}

void HotkeyManager::Start() {
    if (running) return;
    
    running = true;
    
    // Set keyboard hook
    keyboardHook = SetWindowsHookEx(WH_KEYBOARD_LL, KeyboardHook, 
        GetModuleHandle(nullptr), 0);
    
    // Start polling thread (fallback)
    pollThread = std::thread(&HotkeyManager::PollThread, this);
}

void HotkeyManager::PollThread() {
    while (running) {
        for (auto& pair : hotkeys) {
            if (GetAsyncKeyState(pair.first) & 0x8000) {
                DWORD currentTime = GetTickCount();
                if (currentTime - pair.second.lastTrigger > debounceTime) {
                    pair.second.lastTrigger = currentTime;
                    if (pair.second.callback) {
                        pair.second.callback();
                    }
                }
            }
        }
        std::this_thread::sleep_for(std::chrono::milliseconds(10));
    }
}

void HotkeyManager::Stop() {
    running = false;
    
    if (keyboardHook) {
        UnhookWindowsHookEx(keyboardHook);
        keyboardHook = nullptr;
    }
    
    if (pollThread.joinable()) {
        pollThread.join();
    }
}

// hotkey_manager.h
#pragma once
#include <windows.h>
#include <functional>
#include <map>
#include <atomic>
#include <thread>

class HotkeyManager {
private:
    struct HotkeyInfo {
        UINT vkCode;
        std::function<void()> callback;
        DWORD lastTrigger;
    };
    
    std::map<UINT, HotkeyInfo> hotkeys;
    std::atomic<bool> running;
    std::thread pollThread;
    DWORD debounceTime;
    HHOOK keyboardHook;
    
    static LRESULT CALLBACK KeyboardHook(int nCode, WPARAM wParam, LPARAM lParam);
    void PollThread();
    
public:
    HotkeyManager(DWORD debounceMs = 500);
    ~HotkeyManager();
    
    bool RegisterHotkey(UINT vkCode, std::function<void()> callback);
    bool UnregisterHotkey(UINT vkCode);
    void Start();
    void Stop();
};

//mouse.h
#pragma once

#include <windows.h>
#include <string>
#include <vector>
#include <atomic>

class ArduinoMouse {
private:
    HANDLE serial_handle;
    std::string port;
    int filter_length;
    std::vector<int> x_history;
    std::vector<int> y_history;
    std::atomic<bool> connected;
    
    bool FindArduinoPort();
    bool OpenPort(const std::string& portName);
    void WriteData(const unsigned char* data, size_t length);
    void Reconnect();
    void SmoothMove(int x, int y, int& smooth_x, int& smooth_y);
    
public:
    ArduinoMouse(int filter_length = 3);
    ~ArduinoMouse();
    
    bool Connect();
    void Disconnect();
    bool IsConnected() const;
    
    void Move(float x, float y);
    void Flick(float x, float y);
    void Click();
    void Close();
};

//mouse.cpp
#include "mouse.h"
#include <iostream>
#include <chrono>
#include <thread>
#include <algorithm>

ArduinoMouse::ArduinoMouse(int filterLength)
    : filter_length(filterLength),
      serial_handle(INVALID_HANDLE_VALUE),
      connected(false) {
    
    x_history.resize(filter_length, 0);
    y_history.resize(filter_length, 0);
}

ArduinoMouse::~ArduinoMouse() {
    Close();
}

bool ArduinoMouse::FindArduinoPort() {
    // Try common COM ports
    for (int i = 1; i <= 20; ++i) {
        std::string portName = "COM" + std::to_string(i);
        if (OpenPort(portName)) {
            port = portName;
            return true;
        }
    }
    return false;
}

bool ArduinoMouse::OpenPort(const std::string& portName) {
    serial_handle = CreateFileA(
        portName.c_str(),
        GENERIC_READ | GENERIC_WRITE,
        0,
        nullptr,
        OPEN_EXISTING,
        FILE_ATTRIBUTE_NORMAL,
        nullptr
    );
    
    if (serial_handle == INVALID_HANDLE_VALUE) {
        return false;
    }
    
    DCB dcb = { 0 };
    dcb.DCBlength = sizeof(DCB);
    
    if (!GetCommState(serial_handle, &dcb)) {
        CloseHandle(serial_handle);
        serial_handle = INVALID_HANDLE_VALUE;
        return false;
    }
    
    dcb.BaudRate = CBR_115200;
    dcb.ByteSize = 8;
    dcb.StopBits = ONESTOPBIT;
    dcb.Parity = NOPARITY;
    
    if (!SetCommState(serial_handle, &dcb)) {
        CloseHandle(serial_handle);
        serial_handle = INVALID_HANDLE_VALUE;
        return false;
    }
    
    COMMTIMEOUTS timeouts = { 0 };
    timeouts.ReadIntervalTimeout = 50;
    timeouts.ReadTotalTimeoutConstant = 50;
    timeouts.ReadTotalTimeoutMultiplier = 10;
    timeouts.WriteTotalTimeoutConstant = 50;
    timeouts.WriteTotalTimeoutMultiplier = 10;
    
    if (!SetCommTimeouts(serial_handle, &timeouts)) {
        CloseHandle(serial_handle);
        serial_handle = INVALID_HANDLE_VALUE;
        return false;
    }
    
    // Test connection
    unsigned char test = '\n';
    WriteData(&test, 1);
    std::this_thread::sleep_for(std::chrono::milliseconds(100));
    
    connected = true;
    return true;
}

bool ArduinoMouse::Connect() {
    if (FindArduinoPort()) {
        return true;
    }
    
    // Try alternative detection
    std::vector<std::string> commonPorts = { "COM9", "COM3", "COM4", "COM5", "COM6", "COM7", "COM8", "COM10" };
    
    for (const auto& portName : commonPorts) {
        if (OpenPort(portName)) {
            port = portName;
            return true;
        }
    }
    
    return false;
}

void ArduinoMouse::Disconnect() {
    if (serial_handle != INVALID_HANDLE_VALUE) {
        CloseHandle(serial_handle);
        serial_handle = INVALID_HANDLE_VALUE;
    }
    connected = false;
}

bool ArduinoMouse::IsConnected() const {
    return connected;
}

void ArduinoMouse::WriteData(const unsigned char* data, size_t length) {
    if (!connected || serial_handle == INVALID_HANDLE_VALUE) {
        return;
    }
    
    DWORD bytesWritten;
    WriteFile(serial_handle, data, (DWORD)length, &bytesWritten, nullptr);  // Fixed: cast to DWORD
}

void ArduinoMouse::SmoothMove(int x, int y, int& smooth_x, int& smooth_y) {
    // Update history
    x_history.erase(x_history.begin());
    y_history.erase(y_history.begin());
    x_history.push_back(x);
    y_history.push_back(y);
    
    // Calculate average
    smooth_x = 0;
    smooth_y = 0;
    for (int i = 0; i < filter_length; ++i) {
        smooth_x += x_history[i];
        smooth_y += y_history[i];
    }
    smooth_x /= filter_length;
    smooth_y /= filter_length;
}

void ArduinoMouse::Move(float x, float y) {
    if (!connected) {
        Reconnect();
        return;
    }
    
    int smooth_x, smooth_y;
    SmoothMove(static_cast<int>(x), static_cast<int>(y), smooth_x, smooth_y);
    
    // Convert to Arduino format (0-255 with offset for negative)
    int final_x = smooth_x < 0 ? smooth_x + 256 : smooth_x;
    int final_y = smooth_y < 0 ? smooth_y + 256 : smooth_y;
    
    unsigned char data[3] = { 'M', static_cast<unsigned char>(final_x), static_cast<unsigned char>(final_y) };
    WriteData(data, 3);
}

void ArduinoMouse::Flick(float x, float y) {
    if (!connected) {
        Reconnect();
        return;
    }
    
    int final_x = static_cast<int>(x);
    int final_y = static_cast<int>(y);
    
    final_x = final_x < 0 ? final_x + 256 : final_x;
    final_y = final_y < 0 ? final_y + 256 : final_y;
    
    unsigned char data[3] = { 'M', static_cast<unsigned char>(final_x), static_cast<unsigned char>(final_y) };
    WriteData(data, 3);
}

void ArduinoMouse::Click() {
    if (!connected) {
        Reconnect();
        return;
    }
    
    // Add random delay for human-like clicking
    std::this_thread::sleep_for(std::chrono::milliseconds(10 + rand() % 50));
    
    unsigned char data = 'C';
    WriteData(&data, 1);
}

void ArduinoMouse::Reconnect() {
    // Try to reconnect
    Disconnect();
    std::this_thread::sleep_for(std::chrono::seconds(1));
    Connect();
}

void ArduinoMouse::Close() {
    Disconnect();
}

//fov_window.h
#pragma once

#include <opencv2/opencv.hpp>
#include <atomic>
#include <thread>

class Capture;

void ShowDetectionWindow(Capture* grabber, std::atomic<bool>& window_toggled);

//fov_window.cpp
#include "fov_window.h"
#include "capture.h"
#include <chrono>

void ShowDetectionWindow(Capture* grabber, std::atomic<bool>& window_toggled) {
    cv::Scalar LOWER_COLOR = cv::Scalar(140, 110, 150);
    cv::Scalar UPPER_COLOR = cv::Scalar(150, 195, 255);
    
    cv::namedWindow("FOV Window | (Resized For Better View)", cv::WINDOW_NORMAL);
    
    while (window_toggled) {
        cv::Mat screen = grabber->GetScreen();
        if (screen.empty()) {
            std::this_thread::sleep_for(std::chrono::milliseconds(10));
            continue;
        }
        
        cv::Mat hsv;
        cv::cvtColor(screen, hsv, cv::COLOR_BGR2HSV);
        
        cv::Mat mask;
        cv::inRange(hsv, LOWER_COLOR, UPPER_COLOR, mask);
        
        cv::Mat highlighted;
        cv::bitwise_and(screen, screen, highlighted, mask);
        
        cv::Mat blurred;
        cv::GaussianBlur(highlighted, blurred, cv::Size(0, 0), 1, 1);
        
        cv::Mat dimmed;
        screen.copyTo(dimmed);
        dimmed = dimmed * 0.1;
        
        cv::Mat result;
        cv::add(blurred, dimmed, result);
        
        if (result.rows < 500 || result.cols < 500) {
            cv::resize(result, result, cv::Size(500, 500));
        }
        
        cv::imshow("FOV Window | (Resized For Better View)", result);
        
        if (cv::waitKey(1) == 'q') {
            break;
        }
    }
    
    cv::destroyAllWindows();
}


//colorant.h
#pragma once

#include <windows.h>
#include <opencv2/opencv.hpp>
#include <atomic>
#include <thread>
#include <vector>

class Capture;
class ArduinoMouse;

class Colorant {
private:
    ArduinoMouse* mouse;
    Capture* grabber;
    float flickspeed;
    float movespeed;

    std::atomic<bool> toggled;
    std::atomic<bool> window_toggled;
    std::atomic<bool> running;

    cv::Scalar LOWER_COLOR;
    cv::Scalar UPPER_COLOR;

    std::thread listen_thread;

    void ListenThread();
    void ProcessAction(const std::string& action);

public:
    Colorant(int x, int y, int xfov, int yfov, float FLICKSPEED, float MOVESPEED);
    ~Colorant();

    bool Toggle();
    bool IsToggled() const;
    void Close();
};

// Simple factory functions (not DLL exports)
Colorant* CreateColorant(int x, int y, int xfov, int yfov, float FLICKSPEED, float MOVESPEED);
bool ToggleColorant(Colorant* instance);
void CleanupColorant(Colorant* instance);

//capture.h
#pragma once
#include <windows.h>
#include <opencv2/opencv.hpp>
#include <atomic>
#include <thread>
#include <mutex>

class Capture {
private:
    int x, y, xfov, yfov;
    std::atomic<bool> running;
    std::atomic<bool> paused;
    std::thread capture_thread;
    std::mutex frame_mutex;
    cv::Mat current_frame;
    
    void CaptureLoop();
    
public:
    Capture(int x, int y, int xfov, int yfov);
    ~Capture();
    
    cv::Mat GetScreen();
    void Pause();
    void Resume();
    void Stop();
    
    int GetXFov() const { return xfov; }
    int GetYFov() const { return yfov; }
};

//capture.cpp
#include "capture.h"
#include <chrono>

Capture::Capture(int x, int y, int xfov, int yfov)
    : x(x), y(y), xfov(xfov), yfov(yfov),
      running(true), paused(false) {
    
    capture_thread = std::thread(&Capture::CaptureLoop, this);
}

Capture::~Capture() {
    Stop();
}

void Capture::CaptureLoop() {
    HDC hdcScreen = GetDC(nullptr);
    HDC hdcMem = CreateCompatibleDC(hdcScreen);
    HBITMAP hBitmap = CreateCompatibleBitmap(hdcScreen, xfov, yfov);
    SelectObject(hdcMem, hBitmap);
    
    BITMAPINFOHEADER bi;
    bi.biSize = sizeof(BITMAPINFOHEADER);
    bi.biWidth = xfov;
    bi.biHeight = -yfov;  // Negative for top-down
    bi.biPlanes = 1;
    bi.biBitCount = 32;
    bi.biCompression = BI_RGB;
    bi.biSizeImage = 0;
    bi.biXPelsPerMeter = 0;
    bi.biYPelsPerMeter = 0;
    bi.biClrUsed = 0;
    bi.biClrImportant = 0;
    
    auto last_fps_time = std::chrono::steady_clock::now();
    int frame_count = 0;
    
    while (running) {
        if (!paused) {
            // Capture screen
            BitBlt(hdcMem, 0, 0, xfov, yfov, hdcScreen, x, y, SRCCOPY);
            
            // Convert to OpenCV Mat
            cv::Mat mat(yfov, xfov, CV_8UC4);
            GetDIBits(hdcMem, hBitmap, 0, yfov, mat.data, (BITMAPINFO*)&bi, DIB_RGB_COLORS);
            
            // Convert BGRA to BGR
            cv::Mat frame;
            cv::cvtColor(mat, frame, cv::COLOR_BGRA2BGR);
            
            {
                std::lock_guard<std::mutex> lock(frame_mutex);
                current_frame = frame.clone();
            }
            
            // FPS calculation (optional)
            frame_count++;
            auto now = std::chrono::steady_clock::now();
            auto elapsed = std::chrono::duration_cast<std::chrono::seconds>(now - last_fps_time);
            
            if (elapsed.count() >= 1) {
                // Optional: Output FPS to debug
                last_fps_time = now;
                frame_count = 0;
            }
        }
        
        // Small sleep to prevent 100% CPU
        std::this_thread::sleep_for(std::chrono::milliseconds(1));
    }
    
    // Cleanup
    DeleteObject(hBitmap);
    DeleteDC(hdcMem);
    ReleaseDC(nullptr, hdcScreen);
}

cv::Mat Capture::GetScreen() {
    std::lock_guard<std::mutex> lock(frame_mutex);
    return current_frame.clone();
}

void Capture::Pause() {
    paused = true;
}

void Capture::Resume() {
    paused = false;
}

void Capture::Stop() {
    running = false;
    if (capture_thread.joinable()) {
        capture_thread.join();
    }
}



