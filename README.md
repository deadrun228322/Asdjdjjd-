#include <iostream>
#include <vector>
#include <queue>
#include <cmath>
#include <random>
#include <cstdio>
#include <string>
#include <iomanip>

struct Point { double x, y; };

// Проверка попадания в эллипс
bool insideEllipse(double x, double y, double a, double b) {
    return (x*x)/(a*a) + (y*y)/(b*b) <= 1.0;
}

// Равномерная генерация точек внутри эллипса (метод отбраковки)
std::vector<Point> generatePoints(int n, double a, double b, std::mt19937& gen) {
    std::uniform_real_distribution<double> distX(-a, a);
    std::uniform_real_distribution<double> distY(-b, b);
    std::vector<Point> pts;
    pts.reserve(n);
    while ((int)pts.size() < n) {
        double x = distX(gen), y = distY(gen);
        if (insideEllipse(x, y, a, b)) pts.push_back({x, y});
    }
    return pts;
}

// BFS-проверка связности
bool isConnected(const std::vector<std::vector<int>>& adj) {
    int n = adj.size();
    if (n == 0) return true;
    std::vector<bool> vis(n, false);
    std::queue<int> q;
    q.push(0); vis[0] = true;
    int cnt = 0;
    while (!q.empty()) {
        int v = q.front(); q.pop();
        ++cnt;
        for (int u : adj[v]) if (!vis[u]) { vis[u] = true; q.push(u); }
    }
    return cnt == n;
}

// Построение случайного геометрического графа
std::vector<std::vector<int>> buildGraph(const std::vector<Point>& pts, double r) {
    int n = pts.size();
    std::vector<std::vector<int>> adj(n);
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

// Оценка вероятности связности
double connectivityProbability(int n, double a, double c, double r,
                               int trials, std::mt19937& gen) {
    double b = std::sqrt(a*a - c*c);
    int conn = 0;
    for (int t = 0; t < trials; ++t) {
        auto pts = generatePoints(n, a, b, gen);
        auto adj = buildGraph(pts, r);
        if (isConnected(adj)) ++conn;
    }
    return (double)conn / trials;
}

// Вывод характеристик эллипса для каждого c
void printEllipseInfo(double a, const std::vector<double>& cValues) {
    const double PI = std::acos(-1.0);
    std::cout << "\n=== Характеристики эллипса (a фиксирована) ===\n";
    std::cout << std::fixed << std::setprecision(3);
    std::cout << "  a      c        b        e=c/a    S=pi*a*b\n";
    for (double c : cValues) {
        double b = std::sqrt(a*a - c*c);
        double e = c / a;
        double S = PI * a * b;
        std::cout << std::setw(5) << a << "  "
                  << std::setw(6) << c << "  "
                  << std::setw(7) << b << "  "
                  << std::setw(7) << e << "  "
                  << std::setw(10) << S << "\n";
    }
    std::cout << "(c = 0 -> окружность; c -> a -> сильно вытянутый эллипс)\n";
}

int main() {
    // Параметры эксперимента
    const double a       = 10.0;   // большая полуось
    const double cMin    = 0.0;
    const double cMax    = 9.5;    // c < a
    const int    numC    = 20;
    const int    trials  = 300;    // испытаний на точку

    std::random_device rd;
    std::mt19937 gen(rd());

    std::vector<double> cValues;
    for (int i = 0; i < numC; ++i)
        cValues.push_back(cMin + (cMax - cMin) * i / (numC - 1));

    printEllipseInfo(a, cValues);

    // ---------- Эксперимент 1: N фиксировано, меняется r ----------
    const int fixedN = 50;
    std::vector<double> radii = {2.0, 2.5, 3.0, 3.5, 4.0};

    std::cout << "\nЭксперимент 1: N=" << fixedN << ", разные r ...\n";
    std::vector<std::vector<double>> probs1;
    for (double r : radii) {
        std::vector<double> p;
        p.reserve(numC);
        for (double c : cValues)
            p.push_back(connectivityProbability(fixedN, a, c, r, trials, gen));
        probs1.push_back(p);
        std::cout << "  r = " << r << "  готово\n";
    }

    // ---------- Эксперимент 2: r фиксирован, меняется N ----------
    const double fixedR = 3.0;
    std::vector<int> Ns = {20, 40, 60, 80, 100};

    std::cout << "Эксперимент 2: r=" << fixedR << ", разные N ...\n";
    std::vector<std::vector<double>> probs2;
    for (int N : Ns) {
        std::vector<double> p;
        p.reserve(numC);
        for (double c : cValues)
            p.push_back(connectivityProbability(N, a, c, fixedR, trials, gen));
        probs2.push_back(p);
        std::cout << "  N = " << N << "  готово\n";
    }

    // ---------- Отрисовка в gnuplot ----------
    FILE* gp = popen("gnuplot -persist", "w");
    if (!gp) { std::cerr << "gnuplot не найден\n"; return 1; }

    fprintf(gp, "set terminal wxt enhanced size 1500,650 font 'Verdana,10'\n");
    fprintf(gp, "set multiplot layout 1,2 title "
                "'Влияние формы эллипса на вероятность связности графа' "
                "font 'Verdana,12'\n");
    fprintf(gp, "set grid\n");
    fprintf(gp, "set xlabel 'Расстояние между фокусами c  (c=0 -> окружность)'\n");
    fprintf(gp, "set ylabel 'Вероятность связности'\n");
    fprintf(gp, "set yrange [-0.05:1.05]\n");

    // График 1
    fprintf(gp, "set title 'N=%d фиксировано, разные r'\n", fixedN);
    fprintf(gp, "set key outside right\n");
    fprintf(gp, "plot ");
    for (size_t s = 0; s < radii.size(); ++s) {
        if (s) fprintf(gp, ", ");
        fprintf(gp, "'-' with linespoints lw 2 pt 7 title 'r=%.1f'", radii[s]);
    }
    fprintf(gp, "\n");
    for (auto& ser : probs1) {
        for (size_t i = 0; i < cValues.size(); ++i)
            fprintf(gp, "%.4f %.4f\n", cValues[i], ser[i]);
        fprintf(gp, "e\n");
    }

    // График 2
    fprintf(gp, "set title 'r=%.1f фиксировано, разные N'\n", fixedR);
    fprintf(gp, "set key outside right\n");
    fprintf(gp, "plot ");
    for (size_t s = 0; s < Ns.size(); ++s) {
        if (s) fprintf(gp, ", ");
        fprintf(gp, "'-' with linespoints lw 2 pt 7 title 'N=%d'", Ns[s]);
    }
    fprintf(gp, "\n");
    for (auto& ser : probs2) {
        for (size_t i = 0; i < cValues.size(); ++i)
            fprintf(gp, "%.4f %.4f\n", cValues[i], ser[i]);
        fprintf(gp, "e\n");
    }

    fprintf(gp, "unset multiplot\n");
    pclose(gp);

    return 0;
}
