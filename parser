#include <algorithm>
#include <chrono>
#include <conio.h> 
#include <iostream> 
#include <list>
#include <string>
#include <stdio.h>
#include <thread>
#include <vector>
#include <windows.h>
#include <d3d11.h>
#pragma comment(lib, "d3d11.lib")

#include "imgui.h"
#include "imgui_impl_win32.h"
#include "imgui_impl_dx11.h"

bool readLineFromSerial(HANDLE hSerial, std::string& out) {
    char c;
    DWORD bytesRead;
    out.clear();
    while (true) {
        if (!ReadFile(hSerial, &c, 1, &bytesRead, NULL)) return false;
        if (bytesRead == 0) return false; 
        if (c == '\n') break; //Space button
        if (c != '\r') out += c; //Reset button
        Sleep(1);
    }
    return true;
}

// Win32 setup code - From imgui examples and implementation
static ID3D11Device* g_pd3dDevice = NULL;
static ID3D11DeviceContext* g_pd3dDeviceContext = NULL;
static IDXGISwapChain* g_pSwapChain = NULL;
static ID3D11RenderTargetView* g_mainRenderTargetView = NULL;

void CreateRenderTarget()
{
    ID3D11Texture2D* pBackBuffer = NULL;
    g_pSwapChain->GetBuffer(0, IID_PPV_ARGS(&pBackBuffer));
    g_pd3dDevice->CreateRenderTargetView(pBackBuffer, NULL, &g_mainRenderTargetView);
    if (pBackBuffer) pBackBuffer->Release();
}

void CleanupRenderTarget()
{
    if (g_mainRenderTargetView) { g_mainRenderTargetView->Release(); g_mainRenderTargetView = NULL; }
}

bool CreateDeviceD3D(HWND hWnd)
{
    DXGI_SWAP_CHAIN_DESC sd;
    ZeroMemory(&sd, sizeof(sd));
    sd.BufferCount = 2;
    sd.BufferDesc.Format = DXGI_FORMAT_R8G8B8A8_UNORM;
    sd.BufferUsage = DXGI_USAGE_RENDER_TARGET_OUTPUT;
    sd.OutputWindow = hWnd;
    sd.SampleDesc.Count = 1;
    sd.Windowed = TRUE;
    sd.SwapEffect = DXGI_SWAP_EFFECT_DISCARD;

    const D3D_FEATURE_LEVEL featureLevel = D3D_FEATURE_LEVEL_11_0;
    D3D_FEATURE_LEVEL obtained;
    if (D3D11CreateDeviceAndSwapChain(NULL, D3D_DRIVER_TYPE_HARDWARE, NULL, 0,
        &featureLevel, 1, D3D11_SDK_VERSION, &sd, &g_pSwapChain,
        &g_pd3dDevice, &obtained, &g_pd3dDeviceContext) != S_OK)
        return false;

    CreateRenderTarget();
    return true;
}

void CleanupDeviceD3D()
{
    CleanupRenderTarget();
    if (g_pSwapChain) { g_pSwapChain->Release(); g_pSwapChain = NULL; }
    if (g_pd3dDeviceContext) { g_pd3dDeviceContext->Release(); g_pd3dDeviceContext = NULL; }
    if (g_pd3dDevice) { g_pd3dDevice->Release(); g_pd3dDevice = NULL; }
}

// Forward declared in imgui_impl_win32.h
extern LRESULT ImGui_ImplWin32_WndProcHandler(HWND hWnd, UINT msg, WPARAM wParam, LPARAM lParam);

// Global helper to append to ImGui text buffer
static ImGuiTextBuffer g_logBuffer;
static bool g_scrollToBottom = false;

// Append a C-string or std::string to the GUI log
static void appendLog(const char* s)
{
    g_logBuffer.appendf("%s", s);
    g_scrollToBottom = true;
}
static void appendLog(const std::string& s) { appendLog(s.c_str()); }

// Replaces system("cls") clear function
static void clearLog()
{
    g_logBuffer.clear();
    g_scrollToBottom = true;
}

// More Win32 imgui setup code
LRESULT CALLBACK WndProc(HWND hWnd, UINT msg, WPARAM wParam, LPARAM lParam)
{
    if (ImGui_ImplWin32_WndProcHandler(hWnd, msg, wParam, lParam))
        return true;

    switch (msg)
    {
    case WM_SIZE:
        if (g_pSwapChain && wParam != SIZE_MINIMIZED)
        {
            CleanupRenderTarget();
            g_pSwapChain->ResizeBuffers(0, (UINT)LOWORD(lParam), (UINT)HIWORD(lParam), DXGI_FORMAT_UNKNOWN, 0);
            CreateRenderTarget();
        }
        return 0;
    case WM_DESTROY:
        PostQuitMessage(0);
        return 0;
    default:
        return DefWindowProc(hWnd, msg, wParam, lParam);
    }
}

// WinMain setup code
int WINAPI WinMain(HINSTANCE hInstance, HINSTANCE, LPSTR, int)
{
    // Serial port connection
    appendLog("Connecting to COM6...\n");
    HANDLE hSerial = CreateFileA(
        "\\\\.\\COM6", GENERIC_READ, 0, 0, OPEN_EXISTING, 0, 0);
    if (hSerial == INVALID_HANDLE_VALUE) {
        appendLog("Error: Could not open COM6.\n");
        return 1;
    }

    DCB dcbSerialParams = { 0 };
    dcbSerialParams.DCBlength = sizeof(dcbSerialParams);
    if (!GetCommState(hSerial, &dcbSerialParams)) {
        appendLog("Error: Could not get serial parameters.\n");
        CloseHandle(hSerial);
        return 1;
    }

    dcbSerialParams.BaudRate = CBR_9600;
    dcbSerialParams.ByteSize = 8;
    dcbSerialParams.StopBits = ONESTOPBIT;
    dcbSerialParams.Parity = NOPARITY;
    if (!SetCommState(hSerial, &dcbSerialParams)) {
        appendLog("Error: Could not set serial parameters.\n");
        CloseHandle(hSerial);
        return 1;
    }

    COMMTIMEOUTS timeouts = { 0 };
    timeouts.ReadIntervalTimeout = 50;
    timeouts.ReadTotalTimeoutConstant = 50;
    timeouts.ReadTotalTimeoutMultiplier = 10;
    SetCommTimeouts(hSerial, &timeouts);

    appendLog("Connected to COM6 at 9600 baud.\n\n");

    // Draw external display window
    WNDCLASSEX wc = { sizeof(WNDCLASSEX), CS_CLASSDC, WndProc, 0L, 0L,
        GetModuleHandle(NULL), NULL, NULL, NULL, NULL, L"ImGuiMorseWindowClass", NULL };
    RegisterClassEx(&wc);
    HWND hwnd = CreateWindowW(wc.lpszClassName, L"Morse GUI",
        WS_OVERLAPPEDWINDOW, 100, 100, 1280, 800, NULL, NULL, wc.hInstance, NULL);

    if (!CreateDeviceD3D(hwnd)) {
        CleanupDeviceD3D();
        UnregisterClass(wc.lpszClassName, wc.hInstance);
        appendLog("Failed to initialize Direct3D\n");
        CloseHandle(hSerial);
        return 1;
    }

    ShowWindow(hwnd, SW_SHOWDEFAULT);
    UpdateWindow(hwnd);

    // ImGui initialization
    IMGUI_CHECKVERSION();
    ImGui::CreateContext();
    ImGuiIO& io = ImGui::GetIO();
    io.FontGlobalScale = 1.0f;

    ImGui_ImplWin32_Init(hwnd);
    ImGui_ImplDX11_Init(g_pd3dDevice, g_pd3dDeviceContext);

    // Parser logic begins here
    std::list<std::string> word = {};
    std::list<std::string> letter = {};
    std::string input;
    bool shouldDecode = false;
    DWORD lastReceivedTime = GetTickCount(); 

    clearLog();

    bool done = false;
    MSG msg;
    ZeroMemory(&msg, sizeof(msg));
    while (!done)
    {
        // Physical forced exit through keyboard
        while (PeekMessage(&msg, NULL, 0U, 0U, PM_REMOVE))
        {
            TranslateMessage(&msg);
            DispatchMessage(&msg);
            if (msg.message == WM_QUIT) done = true;
        }
        if (done) break;

        // Various input logic
        if (readLineFromSerial(hSerial, input)) {
            if (input == "dot" || input == "dash") { // Normal input
                letter.push_back(input);
                lastReceivedTime = GetTickCount(); 
                appendLog(input + "\n");  
            }
            else if (input == "-") { // Backspace button
                if (!word.empty()) word.pop_back();
                appendLog("\nLAST INPUT REMOVED, PROCEED\n\n");  
            }
            else if (input == "^r") { //Reset button
                letter = {};
                word = {};
            }
            else if (input == "DOT calibrated = ") {
                appendLog("Dot calibrated.\n");
                appendLog("-Input dash-");
            }
            else if (input == "DASH calibrated = ") {
                appendLog("Dash calibrated.");
            }
            else if (input == "CALIBRATION COMPLETE") {
                appendLog("Calibration complete.");
                appendLog("System ready momentarily");
                std::this_thread::sleep_for(std::chrono::seconds(2));
                clearLog();
                appendLog("Enter message:\n");
            }
            else if (input == "/") { // Print button
                appendLog("\n");
                for (const std::string& s : word) appendLog(s);
                appendLog("\n");
                std::list<std::string> sos = { "S","O","S" };
                std::list<std::string> key = { "K","E","Y" };
                std::list<std::string> maze = { "M","A","Z","E" };
                std::list<std::string> elsie = { "E","L","S","I","E" };
                std::list<std::string> east = { "E","A","S","T" };
                std::list<std::string> lewis = { "L","E","W","I","S" };

                if (word == sos) {
                    appendLog("\nDuring WWI, people sent messages through with a telegraph,\nthe machine you are using now!\n\n");
                    appendLog("SOS is a well-known distress signal in morse code\nmeaning -save our souls- or -save our ship-\n");
                    appendLog("This signal was used extensively during WWI by military vessels and aircraft.\n\n");
                    appendLog("Its importance during the Great War was vital \nas the SOS signal alerted rescue teams of downed aircrafts or ships.\n");
                    appendLog("This distress signal saved many lives.\n\n");
                }

                if (word == key) {
                    appendLog("\nAlthough few fully realize, the American Indians were key to the outcome of WWI.\n\n");
                    appendLog("-Many were drafted, but the majority volunteered to fight for America\n\n");
                    appendLog("-Choctaw and Cherokee code-talkers participated in the Meuse-Argonne Offensive in the fall of 1918:\n");
                    appendLog("The key battle that historian Geoffrey Wawro describes as having -cut the German throat-\n\n");
                    appendLog("'The North American Indian took his place beside every other American, \n");
                    appendLog("In offering his life in the great cause, \n");
                    appendLog("Where as a splendid soldier, \n");
                    appendLog("He fought with the courage and valor of his ancestors.'\n");
                    appendLog("General John Pershing\n\n");
                }

                if (word == maze) {
                    appendLog("\nNot many people get a first-hand perspective from soldiers\non a really important aspect of the WWI living situation: Trench Life.\n\n");
                    appendLog("Gilbert Williams, a WWI private, wrote a letter describing his experiences:\n");
                    appendLog(" -Soldiers seem to spend three times as much time in the trenches than out\n");
                    appendLog(" -'a veritable maze of trenches,' where explosives are going off constantly\n");
                    appendLog(" -The fighting was so strong that the dead could only be buried in the sides of the trenches\n\n");
                    appendLog("Trench life was gruesome and hard, yet the soldiers still fought on.\n\n");
                }

                if (word == elsie) {
                    appendLog("The story of Elsie Maud Inglis, although less known,\nhad a huge medical impact in WWI.\n");
                    appendLog("She suggested to the War Office the creation \nof a fully women medical unit on the Western front.\n");
                    appendLog("Initially rejected, she persisted and contacted the French,\n who supported her idea.\n\n");
                    appendLog("Two auxiliary hospitals were sent to Russia and Serbia, \nmajorly subduing the Typhus epidemic.\n\n");
                }

                if (word == east) {
                    appendLog("The Battle of Tannenberg was an influential battle for Germany, \noccuring along their eastern front.\n\n");
                    appendLog("The German military encircled and defeated the Russian Second Army,\n");
                    appendLog("halting the Russian advance into German territory for the rest of WWI.\n");
                    appendLog("They applied these same tactics effectively in future engagements.\n\n");
                    appendLog("With pressure taken off the eastern front,\n");
                    appendLog("Germany could focus more resources on the western front.\n\n");
                    appendLog("The -Tannenberg myth- led to a nationalistic belief in the steadfastness and military prowess\n");
                    appendLog("of the Germans, even after the war had ended.\n\n");
                }

                if (word == lewis) {
                    appendLog("Sidney Lewis was the youngest authenticated British soldier in WW1:\n");
                    appendLog("He enlisted at the age of 12 and fought in the Battle of the Somme in 1916.\n");
                    appendLog("Although, he was eventually sent back home according to the demands of his mom.\n\n");
                    appendLog("In WW1, it was not uncommon for young boys to serve before they turned 18,\n");
                    appendLog("despite the legal enlisting age being 18 years old.\n\n");
                    appendLog("Underage boys might join for various reasons such as:\n");
                    appendLog("patriotism, recruiting pressure, or the desire for escape and adventure.\n\n");
                }

                word.clear();
            }

        }

        // Letter decoding timeout. Changable, within code
        DWORD now = GetTickCount();
        if (!letter.empty() && (now - lastReceivedTime > 2000)) {
            shouldDecode = true;
            appendLog("-Letter time completed: Decoding letter-\n");  
        }

        // Decoding logic
        if (shouldDecode && !letter.empty()) {
            auto it = letter.begin();
            if (it != letter.end() && *it == "dot") {
                ++it;
                if (it != letter.end() && *it == "dot") {
                    ++it;
                    if (it != letter.end() && *it == "dot") {
                        ++it;
                        if (it != letter.end() && *it == "dot") {
                            appendLog("H");
                            word.push_back("H");
                        }
                        else if (it != letter.end() && *it == "dash") {
                            appendLog("V");
                            word.push_back("V");
                        }
                        else {
                            appendLog("S");
                            word.push_back("S");
                        }
                    }
                    else if (it != letter.end() && *it == "dash") {
                        ++it;
                        if (it != letter.end() && *it == "dot") {
                            appendLog("F");
                            word.push_back("F");
                        }
                        else {
                            appendLog("U");
                            word.push_back("U");
                        }
                    }
                    else {
                        appendLog("I");
                        word.push_back("I");
                    }
                }
                else if (it != letter.end() && *it == "dash") {
                    ++it;
                    if (it != letter.end() && *it == "dot") {
                        ++it;
                        if (it != letter.end() && *it == "dot") {
                            appendLog("L");
                            word.push_back("L");
                        }
                        else {
                            appendLog("R");
                            word.push_back("R");
                        }
                    }
                    else if (it != letter.end() && *it == "dash") {
                        ++it;
                        if (it != letter.end() && *it == "dot") {
                            appendLog("P");
                            word.push_back("P");
                        }
                        else if (it != letter.end() && *it == "dash") {
                            appendLog("J");
                            word.push_back("J");
                        }
                        else {
                            appendLog("W");
                            word.push_back("W");
                        }
                    }
                    else {
                        appendLog("A");
                        word.push_back("A");
                    }
                }
                else {
                    appendLog("E");
                    word.push_back("E");
                }
            }
            else if (it != letter.end() && *it == "dash") {
                ++it;
                if (it != letter.end() && *it == "dot") {
                    ++it;
                    if (it != letter.end() && *it == "dot") {
                        ++it;
                        if (it != letter.end() && *it == "dot") {
                            appendLog("B");
                            word.push_back("B");
                        }
                        else if (it != letter.end() && *it == "dash") {
                            appendLog("X");
                            word.push_back("X");
                        }
                        else {
                            appendLog("D");
                            word.push_back("D");
                        }
                    }
                    else if (it != letter.end() && *it == "dash") {
                        ++it;
                        if (it != letter.end() && *it == "dot") {
                            appendLog("C");
                            word.push_back("C");
                        }
                        else if (it != letter.end() && *it == "dash") {
                            appendLog("Y");
                            word.push_back("Y");
                        }
                        else {
                            appendLog("K");
                            word.push_back("K");
                        }
                    }
                    else {
                        appendLog("N");
                        word.push_back("N");
                    }
                }
                else if (it != letter.end() && *it == "dash") {
                    ++it;
                    if (it != letter.end() && *it == "dot") {
                        ++it;
                        if (it != letter.end() && *it == "dot") {
                            appendLog("Z");
                            word.push_back("Z");
                        }
                        else if (it != letter.end() && *it == "dash") {
                            appendLog("Q");
                            word.push_back("Q");
                        }
                        else {
                            appendLog("G");
                            word.push_back("G");
                        }
                    }
                    else if (it != letter.end() && *it == "dash") {
                        appendLog("O");
                        word.push_back("O");
                    }
                    else {
                        appendLog("M");
                        word.push_back("M");
                    }
                }
                else {
                    appendLog("T");
                    word.push_back("T");
                }
            }
            else {
                appendLog("Error decoding input, check values");
            }

            letter.clear();
            shouldDecode = false;
            appendLog("\n");
        }

        // ImGui rendering
        ImGui_ImplDX11_NewFrame();
        ImGui_ImplWin32_NewFrame();
        ImGui::NewFrame();

        // Font scale setup
        ImGui::Begin("Controls");
        ImGui::Text("Font scale:");
        ImGui::SliderFloat("Scale", &io.FontGlobalScale, 0.5f, 3.0f);
        ImGui::SameLine();
        if (ImGui::Button("Clear")) { clearLog(); }
        ImGui::End();

        // Scroll with new input or output
        ImGui::Begin("Serial Output");
        ImGui::BeginChild("ScrollingRegion", ImVec2(0, 0), false, ImGuiWindowFlags_HorizontalScrollbar);
        ImGui::TextUnformatted(g_logBuffer.begin());
        if (g_scrollToBottom) {
            ImGui::SetScrollHereY(1.0f);
            g_scrollToBottom = false;
        }
        ImGui::EndChild();
        ImGui::End();

        // Render Imgui display
        ImGui::Render();
        const float clear_color[4] = { 0.1f, 0.1f, 0.1f, 1.0f };
        g_pd3dDeviceContext->OMSetRenderTargets(1, &g_mainRenderTargetView, NULL);
        g_pd3dDeviceContext->ClearRenderTargetView(g_mainRenderTargetView, clear_color);
        ImGui_ImplDX11_RenderDrawData(ImGui::GetDrawData());
        g_pSwapChain->Present(1, 0); // vsync

        Sleep(1);
    }

    // Imgui exit code
    ImGui_ImplDX11_Shutdown();
    ImGui_ImplWin32_Shutdown();
    ImGui::DestroyContext();

    CleanupDeviceD3D();
    DestroyWindow(hwnd);
    UnregisterClass(wc.lpszClassName, wc.hInstance);

    CloseHandle(hSerial);
    return 0;
}
