# capstone-project-
capstone project wipro 
#include <iostream>
#include <fstream>
#include <string>
#include <sstream>
#include <vector>
#include <unistd.h>
#include <iomanip>
#include <chrono>
#include <thread>
#include <filesystem>
#include <algorithm>
#include <csignal>

using namespace std;
namespace fs = std::filesystem;

struct ProcessInfo {
    int pid;
    string name;
    float cpuUsage;
    float memUsage;
};

// ---------- CPU + Memory ----------
float getCPUUsage() {
    ifstream file("/proc/stat");
    string cpu;
    long user, nice, system, idle, iowait, irq, softirq;
    file >> cpu >> user >> nice >> system >> idle >> iowait >> irq >> softirq;

    static long prevTotal = 0, prevIdle = 0;
    long total = user + nice + system + idle + iowait + irq + softirq;
    long totald = total - prevTotal;
    long idled = idle - prevIdle;
    prevTotal = total;
    prevIdle = idle;
    if (totald == 0) return 0;
    return (float)(totald - idled) / totald * 100.0;
}

float getMemoryUsage() {
    ifstream file("/proc/meminfo");
    string label;
    long totalMem = 1, freeMem = 0;
    while (file >> label) {
        if (label == "MemTotal:") file >> totalMem;
        else if (label == "MemAvailable:") { file >> freeMem; break; }
    }
    return (float)(totalMem - freeMem) / totalMem * 100.0;
}

// ---------- Process Info ----------
vector<ProcessInfo> getProcesses() {
    vector<ProcessInfo> processes;

    for (const auto &entry : fs::directory_iterator("/proc")) {
        if (!entry.is_directory()) continue;

        string pidStr = entry.path().filename();
        if (!all_of(pidStr.begin(), pidStr.end(), ::isdigit))
            continue;

        int pid = stoi(pidStr);
        string commPath = entry.path().string() + "/comm";
        ifstream comm(commPath);
        string name;
        getline(comm, name);
        if (name.empty()) continue;

        string statmPath = entry.path().string() + "/statm";
        ifstream statm(statmPath);
        long size = 0, resident = 0;
        if (!(statm >> size >> resident))
            continue;

        ProcessInfo p;
        p.pid = pid;
        p.name = name;
        p.memUsage = (float)resident / 1000.0;  // approx in MB
        p.cpuUsage = 0.0;                       // placeholder
        processes.push_back(p);
    }

    return processes;
}

// ---------- Display ----------
void display(const vector<ProcessInfo>& processes, float cpu, float mem, const string& sortMode) {
    system("clear");
    cout << "==== Live System Monitor ====\n";
    cout << fixed << setprecision(2);
    cout << "CPU Usage: " << cpu << "% | Memory Usage: " << mem << "% | Sorting by: " << sortMode << "\n";
    cout << string(60, '-') << "\n";
    cout << left << setw(8) << "PID" << setw(25) << "NAME"
         << setw(10) << "MEM(MB)" << setw(10) << "CPU(%)" << "\n";
    cout << string(60, '-') << "\n";

    int count = 0;
    for (auto &p : processes) {
        cout << left << setw(8) << p.pid
             << setw(25) << p.name.substr(0, 24)
             << setw(10) << p.memUsage
             << setw(10) << p.cpuUsage << "\n";
        if (++count > 15) break;
    }
    cout << string(60, '-') << "\n";
    cout << "[Press Ctrl+C to exit]\n";
}

// ---------- Main ----------
int main() {
    string sortMode = "mem";

    while (true) {
        float cpu = getCPUUsage();
        float mem = getMemoryUsage();
        auto processes = getProcesses();

        if (sortMode == "cpu")
            sort(processes.begin(), processes.end(),
                 [](auto &a, auto &b){ return a.cpuUsage > b.cpuUsage; });
        else
            sort(processes.begin(), processes.end(),
                 [](auto &a, auto &b){ return a.memUsage > b.memUsage; });

        display(processes, cpu, mem, sortMode);

        // sleep for 2 seconds before refreshing
        this_thread::sleep_for(chrono::seconds(2));
    }

    return 0;
}
