# 2N49R1M3-powered-by-1NV4S1V0
Script feito por um *construtor civil com* auxilio da **IA Gemini**

Em um momento da minha vida me fiz uma questão sobre os números primos e desde então utilizo a IA no Google Colab pra tentar coletar primos com base em heurísticas que estou tentando criar.

A idéia é que gerando números no formato: **p * p#n até p²** e usando N e a distância de seus fatores p-q com base na *Teoria dos Numeros a densidade média prevista para o local de busca*, o objetivo é ajustar a busca em um porcentagem em relação a N p * q.

## É uma tentativa de uma pessoa simples e comum testar uma curiosidade com o uso da IA Gemini eu cheguei a esse resultado:
```
%%writefile 2N49R1M3.cpp
#include <iostream>
#include <fstream>
#include <gmpxx.h>
#include <vector>
#include <unordered_set>
#include <nlohmann/json.hpp>
#include <thread>
#include <mutex>
#include <csignal>
#include <atomic>
#include <map>
#include <random>
#include <cmath>
#include <algorithm>

using json = nlohmann::json;
std::atomic<bool> keep_running(true);

void signal_handler(int signum) {
    std::cout << "\n[SINAL] Salvando e finalizando motor..." << std::endl;
    keep_running = false;
}

class AdaptiveBrunoEngine {
private:
    std::mutex mtx;
    std::unordered_set<std::string> registry;
    std::map<int, std::vector<mpz_class>> seed_bins;
    json logs = json::array();

    const double PHI = 1.61803398875;
    const double SNAP_DEEP = 0.001;

    double weight_expansion = 0.4;
    std::atomic<int> expansion_hits{0};
    std::atomic<int> harmonic_hits{0};

    void save_memory() {
        std::lock_guard<std::mutex> lock(mtx);
        std::ofstream f("cpi_storage.json");
        if (f.is_open()) {
            f << logs.dump(4);
            f.close();
        }
    }

    void load_memory() {
        std::ifstream f("cpi_storage.json");
        if (f.is_open()) {
            json data;
            try {
                f >> data;
                for (auto& entry : data) {
                    if (entry.contains("v")) {
                        std::string val = entry["v"].get<std::string>();
                        mpz_class p(val);
                        seed_bins[val.length()].push_back(p);
                        registry.insert(val);
                    }
                }
                logs = data;
            } catch (...) {}
        }
        if (registry.empty()) {
            std::vector<unsigned long> base = {2, 3, 5, 7, 11, 13, 17, 19, 23, 29, 31, 37};
            for(auto v : base) seed_bins[1].push_back(mpz_class(v));
        }
    }

    bool is_prime(const mpz_class& c) {
        return mpz_probab_prime_p(c.get_mpz_t(), 25) > 0;
    }

    mpz_class get_random_seed() {
        std::lock_guard<std::mutex> lock(mtx);
        if (seed_bins.empty()) return mpz_class(2);

        // SELETOR INTELIGENTE BALANCEADO
        // 70% de chance de focar em magnitudes férteis (Elite Dinâmica)
        if (rand() % 100 < 70) {
            std::vector<int> magnitudes;
            for(auto const& [mag, list] : seed_bins) magnitudes.push_back(mag);

            // Ordena para encontrar as magnitudes com mais sementes
            std::sort(magnitudes.begin(), magnitudes.end(), [&](int a, int b) {
                return seed_bins[a].size() > seed_bins[b].size();
            });

            // Escolhe entre as 3 melhores magnitudes para não ficar preso em uma só
            int top_n = std::min((int)magnitudes.size(), 3);
            int chosen_mag = magnitudes[rand() % top_n];
            return seed_bins[chosen_mag][rand() % seed_bins[chosen_mag].size()];
        }

        // 30% de exploração aleatória da malha (Mesh Exploration)
        auto it = seed_bins.begin();
        std::advance(it, rand() % seed_bins.size());
        return it->second[rand() % it->second.size()];
    }

    void record_hit(const mpz_class& cand, const std::string& h_type) {
        std::string sv = cand.get_str();
        std::lock_guard<std::mutex> lock(mtx);
        if (registry.find(sv) == registry.end()) {
            registry.insert(sv);
            seed_bins[sv.length()].push_back(cand);
            logs.push_back({{"v", sv}, {"type", h_type}, {"mag", sv.length()}});
            std::cout << "[HIT] " << h_type << " | Mag: " << sv.length() << " | " << (sv.length() > 20 ? sv.substr(0, 20) + "..." : sv) << std::endl;
        }
    }

public:
    AdaptiveBrunoEngine() { load_memory(); }

    void run_cycle(int thread_id) {
        std::random_device rd;
        std::mt19937 gen(rd());
        std::uniform_real_distribution<> dis(0.0, 1.0);
        long current_delta = 300;
        int local_cycles = 0;

        while (keep_running) {
            local_cycles++;
            mpz_class p = get_random_seed();

            mpz_class m1 = 2 * p - 1, m2 = 2 * p + 1;
            if (is_prime(m1)) record_hit(m1, "na_mira");
            if (is_prime(m2)) record_hit(m2, "na_mira");

            double current_limit = std::max(SNAP_DEEP, weight_expansion);
            bool use_expansion = (dis(gen) < current_limit);
            mpz_class anchor;
            std::string h_type;

            if (use_expansion) {
                unsigned long E = 50 + (rand() % 5000);
                mpz_class base_exp; mpz_pow_ui(base_exp.get_mpz_t(), mpz_class(2).get_mpz_t(), E);
                anchor = p * base_exp;
                h_type = "expansion_2E";
            } else {
                mpz_class pn = 1, pg = 2;
                int depth = 3 + (rand() % 30);
                for(int i=0; i<depth; ++i) { pn *= pg; mpz_nextprime(pg.get_mpz_t(), pg.get_mpz_t()); }
                anchor = p * pn;
                h_type = "harmonic_p#n";
            }

            for (long i = -current_delta; i <= current_delta && keep_running; ++i) {
                mpz_class cand = anchor + i;
                if (is_prime(cand)) {
                    record_hit(cand, h_type);
                    if (use_expansion) expansion_hits++; else harmonic_hits++;
                    current_delta = std::min((long)15000, (long)(current_delta * PHI));
                }
            }
            if (local_cycles % 15 == 0) {
                weight_expansion = (double(expansion_hits + 1) / (expansion_hits + harmonic_hits + 2));
                save_memory();
            }
        }
    }

    void start() {
        std::vector<std::thread> threads;
        int n = std::thread::hardware_concurrency();
        for (int i = 0; i < n; ++i) threads.emplace_back(&AdaptiveBrunoEngine::run_cycle, this, i);
        for (auto& t : threads) t.join();
    }
};

int main() {
    srand(time(0));
    signal(SIGINT, signal_handler);
    AdaptiveBrunoEngine motor;
    motor.start();
    return 0;
}

```

## *Compilando em Jupiter notebook via Google Colab:*
```
!g++ -O3 2N49R1M3.cpp -o 2N49R1M3 -lgmpxx -lgmp
print("--- Motor 2N49R1M3: Inteligência de Semente Atualizada ---")
```

Atualmente eu utilizo o Drive para colerar primos e futuramente analisar.

link: ```https://drive.google.com/uc?export=download&id=1Mpe1xHlg50uYvF-1ZqKrWfUtdin2KsGl```
