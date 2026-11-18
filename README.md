// ellipse_graph_windows.cpp
// Компилятор: MSVC или MinGW-w64
#define _CRT_SECURE_NO_WARNINGS
#define _USE_MATH_DEFINES
#define UNICODE
#define _UNICODE
#define NOMINMAX

#include <windows.h>
#include <gdiplus.h>
#include <shellapi.h>
#include <iostream>
#include <vector>
#include <queue>
#include <cmath>
#include <random>
#include <string>
#include <sstream>
#include <iomanip>
#include <clocale>

#pragma comment(lib, "gdiplus.lib")
#pragma comment(lib, "shell32.lib")
#pragma comment(lib, "gdi32.lib")

using namespace Gdiplus;
using namespace std;

// ==================== Моделирование ====================

struct Point { double x, y; };

bool insideEllipse(double x, double y, double a, double b) {
    return (x*x)/(a*a) + (y*y)/(b*b) <= 1.0;
}

vector<Point> generatePoints(int n, double a, double b, mt19937& gen) {
    uniform_real_distribution<double> dx(-a, a), dy(-b, b);
    vector<Point> pts; pts.reserve(n);
    while ((int)pts.size() < n) {
        double x = dx(gen), y = dy(gen);
        if (insideEllipse(x, y, a, b)) pts.push_back({x, y});
    }
    return pts;
}

bool isConnected(const vector<vector<int>>& adj) {
    int n = adj.size();
    if (n == 0) return true;
    vector<bool> vis(n, false);
    queue<int> q; q.push(0); vis[0] = true;
    int cnt = 0;
    while (!q.empty()) {
        int v = q.front(); q.pop(); ++cnt;
        for (int u : adj[v]) if (!vis[u]) { vis[u] = true; q.push(u); }
    }
    return cnt == n;
}

vector<vector<int>> buildGraph(const vector<Point>& pts, double r) {
    int n = pts.size();
    vector<vector<int>> adj(n);
    for (int i = 0; i < n; ++i)
        for (int j = i+1; j < n; ++j) {
            double dx = pts[i].x - pts[j].x;
            double dy = pts[i].y - pts[j].y;
            if (dx*dx + dy*dy <= r*r) {
                adj[i].push_back(j);
                adj[j].push_back(i);
            }
        }
    return adj;
}

double connectivityProbability(int n, double a, double c, double r,
                               int trials, mt19937& gen) {
    double b = sqrt(a*a - c*c);
    int conn = 0;
    for (int t = 0; t < trials; ++t) {
        auto pts = generatePoints(n, a, b, gen);
        auto adj = buildGraph(pts, r);
        if (isConnected(adj)) ++conn;
    }
    return (double)conn / trials;
}

// ==================== Рисование через GDI+ ====================

struct PlotSpec {
    wstring title, xlabel, ylabel;
    vector<double> xs;
    vector<vector<double>> series;
    vector<wstring> labels;
};

void drawPlot(Graphics& g, const RectF& area, const PlotSpec& spec) {
    SolidBrush bg(Color(255, 250, 250, 252));
    g.FillRectangle(&bg, area);

    float left   = area.X + 80;
    float right  = area.GetRight() - 150;
    float top    = area.Y + 60;
    float bottom = area.GetBottom() - 75;
    float w = right - left, h = bottom - top;
    if (w <= 0 || h <= 0) return;

    double xmin = spec.xs.empty() ? 0.0 : spec.xs.front();
    double xmax = spec.xs.empty() ? 1.0 : spec.xs.back();
    if (xmax == xmin) xmax = xmin + 1.0;

    // Сетка
    Pen grid(Color(100, 210, 210, 220), 1.0f);
    grid.SetDashStyle(DashStyleDot);
    for (int i = 0; i <= 5; i++) {
        float y = bottom - i * h / 5;
        g.DrawLine(&grid, left, y, right, y);
        float x = left + i * w / 5;
        g.DrawLine(&grid, x, top, x, bottom);
    }

    // Оси
    Pen axis(Color(255, 50, 50, 50), 1.5f);
    g.DrawLine(&axis, left, top, left, bottom);
    g.DrawLine(&axis, left, bottom, right, bottom);

    FontFamily ff(L"Arial");
    Font fAxis(&ff, 12, FontStyleRegular, UnitPixel);
    Font fTitle(&ff, 17, FontStyleBold, UnitPixel);
    Font fLegend(&ff, 11, FontStyleRegular, UnitPixel);
    SolidBrush tb(Color(255, 30, 30, 30));

    // Подписи по Y
    for (int i = 0; i <= 5; i++) {
        float y = bottom - i * h / 5;
        wostringstream wss;
        wss << fixed << setprecision(1) << (i * 0.2);
        g.DrawString(wss.str().c_str(), -1, &fAxis, PointF(left - 42, y - 9), &tb);
    }
    // Подписи по X
    for (int i = 0; i <= 5; i++) {
        float x = left + i * w / 5;
        double xv = xmin + (xmax - xmin) * i / 5.0;
        wostringstream wss;
        wss << fixed << setprecision(1) << xv;
        g.DrawString(wss.str().c_str(), -1, &fAxis, PointF(x - 15, bottom + 8), &tb);
    }

    Color colors[] = {
        Color(255, 220, 50, 50),  Color(255, 50, 150, 50),
        Color(255, 50, 80, 220),  Color(255, 230, 140, 30),
        Color(255, 150, 50, 180), Color(255, 30, 150, 150)
    };

    // Кривые
    for (size_t s = 0; s < spec.series.size(); s++) {
        Pen pen(colors[s % 6], 2.5f);
        SolidBrush dot(colors[s % 6]);
        const auto& ys = spec.series[s];
        PointF prev; bool first = true;
        for (size_t i = 0; i < spec.xs.size() && i < ys.size(); i++) {
            float px = left + (float)((spec.xs[i] - xmin) / (xmax - xmin) * w);
            float py = bottom - (float)(ys[i] * h);
            if (py < top) py = top;
            if (py > bottom) py = bottom;
            PointF cur(px, py);
            if (!first) g.DrawLine(&pen, prev, cur);
            g.FillEllipse(&dot, px - 3.5f, py - 3.5f, 7.0f, 7.0f);
            prev = cur; first = false;
        }
    }

    StringFormat center;
    center.SetAlignment(StringAlignmentCenter);
    center.SetLineAlignment(StringAlignmentCenter);

    // Заголовок
    g.DrawString(spec.title.c_str(), -1, &fTitle,
                 RectF(area.X + 20, area.Y + 12, area.Width - 40, 32),
                 &center, &tb);

    // Легенда
    for (size_t s = 0; s < spec.labels.size(); s++) {
        float ly = top + 10 + s * 22.0f;
        Pen pen(colors[s % 6], 2.5f);
        g.DrawLine(&pen, right + 15, ly + 6, right + 45, ly + 6);
        g.DrawString(spec.labels[s].c_str(), -1, &fLegend,
                     PointF(right + 50, ly - 2), &tb);
    }

    // Подпись X
    g.DrawString(spec.xlabel.c_str(), -1, &fAxis,
                 RectF(area.X, bottom + 32, area.Width, 25), &center, &tb);

    // Подпись Y (повёрнутая)
    Matrix m;
    m.RotateAt(-90, PointF(area.X + 22, (top + bottom) / 2));
    g.SetTransform(&m);
    g.DrawString(spec.ylabel.c_str(), -1, &fAxis,
                 RectF(area.X - 130, (top + bottom) / 2 - 15, 260, 30),
                 &center, &tb);
    g.ResetTransform();

    // Рамка
    Pen border(Color(255, 150, 150, 150), 1.0f);
    g.DrawRectangle(&border, area);
}

bool renderTwoPlots(const wstring& filename,
                    const PlotSpec& s1, const PlotSpec& s2) {
    int W = 1500, H = 720;
    Bitmap bmp(W, H, PixelFormat32bppARGB);
    Graphics g(&bmp);
    g.SetSmoothingMode(SmoothingModeAntiAlias);
    g.SetTextRenderingHint(TextRenderingHintAntiAlias);

    SolidBrush white(Color(255, 255, 255, 255));
    g.FillRectangle(&white, 0, 0, W, H);

    drawPlot(g, RectF(5, 5, W/2 - 10, H - 10), s1);
    drawPlot(g, RectF(W/2 + 5, 5, W/2 - 10, H - 10), s2);

    // Найти PNG-энкодер и сохранить
    CLSID png;
    UINT n = 0, sz = 0;
    GetImageEncodersSize(&n, &sz);
    ImageCodecInfo* codecs = (ImageCodecInfo*)malloc(sz);
    GetImageEncoders(n, sz, codecs);
    bool found = false;
    for (UINT i = 0; i < n; i++)
        if (wcscmp(codecs[i].MimeType, L"image/png") == 0) {
            png = codecs[i].Clsid; found = true; break;
        }
    free(codecs);
    if (!found) return false;

    return bmp.Save(filename.c_str(), &png, NULL) == Ok;
}

// ==================== main ====================

int main() {
    SetConsoleOutputCP(CP_UTF8);
    setlocale(LC_ALL, "");

    GdiplusStartupInput gsi;
    ULONG_PTR token;
    GdiplusStartup(&token, &gsi, NULL);

    const double a     = 10.0;
    const double cMin  = 0.0, cMax = 9.5;
    const int    numC  = 20;
    const int    trials = 300;

    random_device rd;
    mt19937 gen(rd());

    vector<double> cValues;
    for (int i = 0; i < numC; ++i)
        cValues.push_back(cMin + (cMax - cMin) * i / (numC - 1));

    cout << "=== Характеристики эллипса (a = " << a << ") ===\n";
    cout << fixed << setprecision(3);
    cout << "   c        b       e=c/a      S=pi*a*b\n";
    for (double c : cValues) {
        double b = sqrt(a*a - c*c);
        cout << setw(5) << c << "  " << setw(8) << b << "  "
             << setw(7) << c/a << "  " << setw(11)
             << M_PI * a * b << "\n";
    }

    // Эксперимент 1: фиксированное N, разные r
    const int fixedN = 50;
    vector<double> radii = {2.0, 2.5, 3.0, 3.5, 4.0};
    cout << "\nЭксперимент 1: N=" << fixedN << ", разные r...\n";
    vector<vector<double>> p1; vector<wstring> l1;
    for (double r : radii) {
        vector<double> p;
        for (double c : cValues)
            p.push_back(connectivityProbability(fixedN, a, c, r, trials, gen));
        p1.push_back(p);
        wostringstream wss; wss << L"r = " << fixed << setprecision(1) << r;
        l1.push_back(wss.str());
        cout << "  r = " << r << " готово\n";
    }

    // Эксперимент 2: фиксированный r, разные N
    const double fixedR = 3.0;
    vector<int> Ns = {20, 40, 60, 80, 100};
    cout << "\nЭксперимент 2: r=" << fixedR << ", разные N...\n";
    vector<vector<double>> p2; vector<wstring> l2;
    for (int N : Ns) {
        vector<double> p;
        for (double c : cValues)
            p.push_back(connectivityProbability(N, a, c, fixedR, trials, gen));
        p2.push_back(p);
        wostringstream wss; wss << L"N = " << N;
        l2.push_back(wss.str());
        cout << "  N = " << N << " готово\n";
    }

    // Собираем PlotSpec
    PlotSpec s1{L"N фиксировано, меняется r",
                L"Расстояние между фокусами c",
                L"Вероятность связности",
                cValues, p1, l1};

    PlotSpec s2{L"r фиксировано, меняется N",
                L"Расстояние между фокусами c",
                L"Вероятность связности",
                cValues, p2, l2};

    if (renderTwoPlots(L"graphs.png", s1, s2)) {
        cout << "\nГотово: graphs.png создан и открыт.\n";
        ShellExecuteW(NULL, L"open", L"graphs.png", NULL, NULL, SW_SHOWNORMAL);
    } else {
        cerr << "\nОшибка сохранения graphs.png\n";
    }

    GdiplusShutdown(token);
    return 0;
}
