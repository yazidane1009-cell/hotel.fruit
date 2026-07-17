# hotel.fruit
/**
 * @license
 * SPDX-License-Identifier: Apache-2.0
 */

import React, { useState, useEffect, useMemo } from 'react';
import { FruitType, LogEntry, InventoryItem, DEFAULT_FRUITS } from './types';
import { generateMockData, exportToCSV, exportToJSON, formatCurrency } from './utils';
import MetricCard from './components/MetricCard';
import FruitForm from './components/FruitForm';
import InventorySummary from './components/InventorySummary';
import DashboardCharts, { DataBreakdownChart } from './components/DashboardCharts';
import LogsTable from './components/LogsTable';
import PeriodicStats from './components/PeriodicStats';
import FruitUsageAnalysis from './components/FruitUsageAnalysis';
import { 
  Apple, 
  Banana,
  Grape,
  Citrus,
  Cherry,
  Trash2, 
  RefreshCw, 
  Database, 
  Sparkles, 
  ShieldCheck, 
  Clock, 
  Building2,
  Settings,
  HelpCircle,
  CalendarDays,
  Activity,
  History
} from 'lucide-react';
import { motion } from 'motion/react';

export default function App() {
  // ----------------------------------------------------
  // 1. STATE INITIALIZATION & LOCALSTORAGE SYNC
  // ----------------------------------------------------
  
  // Tab control: 'operations' (live entry + charts), 'periodic' (day/week/month/quarter/year stats), 'analysis' (usage/loss analytics), 'history' (independent logs and stock)
  const [activeTab, setActiveTab] = useState<'operations' | 'periodic' | 'analysis' | 'history'>('operations');

  // Load Fruits list or default
  const [fruits, setFruits] = useState<FruitType[]>(() => {
    const saved = localStorage.getItem('hotel_fruits');
    if (saved) {
      try {
        return JSON.parse(saved);
      } catch (e) {
        console.error('Failed to parse saved fruits', e);
      }
    }
    return [...DEFAULT_FRUITS];
  });

  // Load Logs list. If empty, seed mock data so the app starts gorgeous!
  const [logs, setLogs] = useState<LogEntry[]>(() => {
    const saved = localStorage.getItem('hotel_fruit_logs');
    if (saved) {
      try {
        const parsed = JSON.parse(saved);
        if (parsed && Array.isArray(parsed) && parsed.length > 0) {
          return parsed;
        }
      } catch (e) {
        console.error('Failed to parse saved logs', e);
      }
    }
    // Seed gorgeous mock data immediately on first load
    const { logs: mockLogs } = generateMockData();
    return mockLogs;
  });

  // Save states to LocalStorage on changes
  useEffect(() => {
    localStorage.setItem('hotel_fruits', JSON.stringify(fruits));
  }, [fruits]);

  useEffect(() => {
    localStorage.setItem('hotel_fruit_logs', JSON.stringify(logs));
  }, [logs]);

  // ----------------------------------------------------
  // 2. REAL-TIME DERIVED STATE CALCULATIONS
  // ----------------------------------------------------

  // Calculate real-time current lobby stock per fruit
  const inventory = useMemo(() => {
    const inv: Record<string, number> = {};
    
    // Initialize
    fruits.forEach(f => {
      inv[f.id] = 0;
    });

    // Sort logs chronologically to compute real-time state safely
    const sortedLogs = [...logs].sort((a, b) => a.timestamp - b.timestamp);

    sortedLogs.forEach(log => {
      if (inv[log.fruitId] === undefined) {
        inv[log.fruitId] = 0; // fallback for newly added fruits
      }

      if (log.type === 'supply') {
        inv[log.fruitId] += log.quantity;
      } else if (log.type === 'consumption') {
        inv[log.fruitId] -= log.quantity;
      } else if (log.type === 'wastage') {
        inv[log.fruitId] -= log.quantity;
      }
    });

    return inv;
  }, [logs, fruits]);

  // Calculate aggregated totals (supplied, consumed, wasted) per fruit
  const totals = useMemo(() => {
    const counts: Record<string, InventoryItem> = {};
    
    fruits.forEach(f => {
      counts[f.id] = {
        fruitId: f.id,
        currentDisplay: inventory[f.id] || 0,
        totalSupplied: 0,
        totalConsumed: 0,
        totalWasted: 0
      };
    });

    logs.forEach(log => {
      if (!counts[log.fruitId]) {
        counts[log.fruitId] = {
          fruitId: log.fruitId,
          currentDisplay: inventory[log.fruitId] || 0,
          totalSupplied: 0,
          totalConsumed: 0,
          totalWasted: 0
        };
      }

      if (log.type === 'supply') {
        counts[log.fruitId].totalSupplied += log.quantity;
      } else if (log.type === 'consumption') {
        counts[log.fruitId].totalConsumed += log.quantity;
      } else if (log.type === 'wastage') {
        counts[log.fruitId].totalWasted += log.quantity;
      }
    });

    return counts;
  }, [logs, fruits, inventory]);

  // Calculate Global KPI Stats
  const globalMetrics = useMemo(() => {
    let suppliedQty = 0;
    let suppliedCost = 0;
    
    let consumedQty = 0;
    let consumedCost = 0;
    
    let wastedQty = 0;
    let wastedCost = 0;

    logs.forEach(log => {
      if (log.type === 'supply') {
        suppliedQty += log.quantity;
        suppliedCost += log.cost;
      } else if (log.type === 'consumption') {
        consumedQty += log.quantity;
        consumedCost += log.cost;
      } else if (log.type === 'wastage') {
        wastedQty += log.quantity;
        wastedCost += log.cost;
      }
    });

    // Waste rate = wasted / (consumed + wasted) or wasted / supplied. Let's use wasted / supplied as standard
    const wasteRate = suppliedQty > 0 ? (wastedQty / suppliedQty) * 100 : 0;
    const activeDisplayQty = suppliedQty - consumedQty - wastedQty;

    return {
      suppliedQty,
      suppliedCost,
      consumedQty,
      consumedCost,
      wastedQty,
      wastedCost,
      wasteRate,
      activeDisplayQty
    };
  }, [logs]);

  // ----------------------------------------------------
  // 3. ACTION HANDLERS
  // ----------------------------------------------------

  // Create a new log entry
  const handleLogSubmit = (newLog: Omit<LogEntry, 'id' | 'cost'> & { id?: string; cost?: number }) => {
    const fruit = fruits.find(f => f.id === newLog.fruitId);
    if (!fruit) return;

    const logId = `log-${Date.now()}-${Math.floor(Math.random() * 1000)}`;
    const costValue = newLog.cost !== undefined ? newLog.cost : fruit.defaultCost * newLog.quantity;

    const entry: LogEntry = {
      ...newLog,
      id: logId,
      cost: costValue
    };

    setLogs(prev => [entry, ...prev]);
  };

  // Delete an entry
  const handleDeleteLog = (id: string) => {
    setLogs(prev => prev.filter(log => log.id !== id));
  };

  // Add custom fruit type
  const handleAddFruit = (newFruit: Omit<FruitType, 'id'> & { id: string }) => {
    setFruits(prev => [...prev, newFruit]);
  };

  // Edit fruit default pricing
  const handleUpdateFruitPrice = (id: string, newCost: number) => {
    setFruits(prev => prev.map(f => f.id === id ? { ...f, defaultCost: newCost } : f));
    
    // Also optionally update costs in existing logs for consistency, or leave historical logs as-is.
    // Standard bookkeeping practice is to leave historical logs as-is (reflecting cost at time of entry).
    // We will leave historical as-is!
  };

  // Wipe out everything
  const handleClearAllData = () => {
    setLogs([]);
    setFruits([...DEFAULT_FRUITS]);
    localStorage.removeItem('hotel_fruit_logs');
    localStorage.removeItem('hotel_fruits');
  };

  // Re-seed simulation data
  const handleSeedDemoData = () => {
    const { logs: mockLogs, fruits: mockFruits } = generateMockData();
    setFruits(mockFruits);
    setLogs(mockLogs);
  };

  // Export handlers
  const handleExportCSV = () => {
    exportToCSV(logs, fruits);
  };

  const handleExportJSON = () => {
    exportToJSON(logs, fruits);
  };

  // Import Backup from JSON file
  const handleImportJSON = (event: React.ChangeEvent<HTMLInputElement>) => {
    const file = event.target.files?.[0];
    if (!file) return;

    const reader = new FileReader();
    reader.onload = (e) => {
      try {
        const data = JSON.parse(e.target?.result as string);
        if (data && Array.isArray(data.logs) && Array.isArray(data.fruits)) {
          setFruits(data.fruits);
          setLogs(data.logs);
          alert('🎉 備份資料已成功匯入並覆蓋目前紀錄！');
        } else {
          alert('❌ 備份檔格式不正確，必須包含水果(fruits)與紀錄(logs)陣列。');
        }
      } catch (err) {
        alert('❌ 讀取備份檔案失敗，請確保此為正確的 JSON 備份檔。');
      }
    };
    reader.readAsText(file);
    // Reset file input value so same file can be imported again if needed
    event.target.value = '';
  };

  // ----------------------------------------------------
  // 4. RENDER LAYOUT
  // ----------------------------------------------------
  return (
    <div className="min-h-screen bg-slate-50/60 font-sans text-slate-800 antialiased selection:bg-indigo-100 selection:text-indigo-900 pb-16">
      
      {/* Dynamic Grid Header & Banner */}
      <header className="sticky top-0 z-40 border-b border-slate-100 bg-white/90 backdrop-blur-md">
        <div className="mx-auto max-w-7xl px-4 sm:px-6 lg:px-8">
          <div className="flex h-16 items-center justify-between gap-4">
            
            {/* Title Block */}
            <div className="flex items-center gap-2.5">
              <div className="flex h-10 w-10 items-center justify-center rounded-xl bg-emerald-500 text-white shadow-sm shadow-emerald-200">
                <Apple className="h-5.5 w-5.5 fill-current" />
              </div>
              <div>
                <h1 className="text-md font-bold text-slate-900 sm:text-lg tracking-tight">迎賓水果統計系統</h1>
                <p className="hidden text-[11px] text-slate-400 sm:block font-medium">Lobby Welcome Fruits Analytics & Waste Tracker</p>
              </div>
            </div>

            {/* Quick Context Block (Hotel Info, Datetime) */}
            <div className="flex items-center gap-3">
              <div className="hidden flex-col items-end text-xs md:flex">
                <span className="flex items-center gap-1 font-semibold text-slate-700">
                  <Building2 className="h-3.5 w-3.5 text-slate-400" />
                  大廳迎賓櫃檯
                </span>
                <span className="flex items-center gap-1 text-[10px] text-slate-400 font-medium mt-0.5">
                  <Clock className="h-3.5 w-3.5 text-slate-400" />
                  系統時間：2026-07-16
                </span>
              </div>
              
              <div className="h-8 w-px bg-slate-100 hidden md:block" />

              {/* Data Safety Indicator */}
              <span className="inline-flex items-center gap-1 rounded-full bg-emerald-50 px-2.5 py-1 text-[10px] font-bold text-emerald-700 border border-emerald-100">
                <ShieldCheck className="h-3 w-3 shrink-0" />
                瀏覽器儲存模式
              </span>
            </div>

          </div>
        </div>
      </header>

      {/* Main Container */}
      <main className="mx-auto max-w-7xl px-4 pt-6 sm:px-6 lg:px-8 space-y-6">
        
        {/* Row 1: KPI Stats Grid */}
        <section id="kpi_stats_grid" className="grid grid-cols-2 gap-4 md:grid-cols-4 lg:grid-cols-5">
          <MetricCard
            id="metric_total_supplied"
            title="累積補貨量"
            value={`${globalMetrics.suppliedQty.toLocaleString()}`}
            subValue={`總額 ${formatCurrency(globalMetrics.suppliedCost)}`}
            iconName="PlusCircle"
            colorClass="emerald"
            index={0}
          />

          <MetricCard
            id="metric_total_consumed"
            title="旅客食用數量"
            value={`${globalMetrics.consumedQty.toLocaleString()}`}
            subValue={`食用佔比 ${globalMetrics.suppliedQty > 0 ? Math.round((globalMetrics.consumedQty / globalMetrics.suppliedQty) * 100) : 0}%`}
            subValueColor="text-indigo-600"
            iconName="ShoppingBag"
            colorClass="indigo"
            index={1}
          />

          <MetricCard
            id="metric_total_wasted"
            title="報廢數量"
            value={`${globalMetrics.wastedQty.toLocaleString()}`}
            subValue={`損失金額 ${formatCurrency(globalMetrics.wastedCost)}`}
            subValueColor="text-rose-600"
            iconName="Trash2"
            colorClass="rose"
            index={2}
          />

          <MetricCard
            id="metric_waste_rate"
            title="報廢比率"
            value={`${globalMetrics.wasteRate.toFixed(1)}%`}
            subValue={globalMetrics.wasteRate > 20 ? '🚨 報廢率偏高需調整' : '✓ 報廢率符合綠色標準'}
            subValueColor={globalMetrics.wasteRate > 20 ? 'text-rose-600 font-bold' : 'text-emerald-600'}
            iconName="AlertTriangle"
            colorClass={globalMetrics.wasteRate > 20 ? 'rose' : globalMetrics.wasteRate > 10 ? 'amber' : 'sky'}
            index={3}
          />

          {/* Current display count */}
          <div className="col-span-2 md:col-span-4 lg:col-span-1 rounded-2xl border border-indigo-100 bg-indigo-50/20 p-6 flex flex-col justify-between shadow-sm relative overflow-hidden">
            <div className="absolute -right-6 -bottom-6 text-indigo-100/50">
              <Apple className="h-24 w-24 fill-current rotate-12" />
            </div>
            
            <div className="z-10">
              <p className="text-xs font-bold text-indigo-700 uppercase tracking-wide">目前大廳水果</p>
              <h3 className="text-3xl font-black text-slate-900 mt-1.5">
                {globalMetrics.activeDisplayQty} <span className="text-sm font-semibold text-slate-500">個</span>
              </h3>
              <p className="text-[10px] text-slate-500 mt-1">現有擺放估計價值: {formatCurrency(globalMetrics.suppliedCost - globalMetrics.consumedCost - globalMetrics.wastedCost)}</p>
            </div>
            
            <div className="mt-4 border-t border-indigo-100/50 pt-2 z-10 flex gap-2">
              <button
                id="reset_simulation_data_btn"
                onClick={handleSeedDemoData}
                className="flex items-center gap-1 rounded-lg bg-indigo-600 px-2.5 py-1.5 text-[10px] font-bold text-white hover:bg-indigo-700 transition cursor-pointer"
                title="重置為包含14天歷史數據的範例"
              >
                <Sparkles className="h-3 w-3 shrink-0" />
                載入模擬數據
              </button>
              
              <button
                id="clear_all_data_btn"
                onClick={() => {
                  if (confirm('⚠️ 您確定要清空所有數據嗎？此操作將會刪除您所有的本地紀錄，無法還原！')) {
                    handleClearAllData();
                  }
                }}
                className="flex items-center gap-1 rounded-lg bg-white border border-rose-200 px-2.5 py-1.5 text-[10px] font-bold text-rose-600 hover:bg-rose-50 transition cursor-pointer"
                title="清空所有歷史紀錄重來"
              >
                <Trash2 className="h-3 w-3 shrink-0" />
                清空重置
              </button>
            </div>
          </div>
        </section>

        {/* Navigation Tabs */}
        <div id="dashboard_tabs" className="flex border-b border-slate-200 overflow-x-auto scrollbar-none">
          <button
            id="tab_ops"
            onClick={() => setActiveTab('operations')}
            className={`flex items-center gap-2 border-b-2 px-6 py-3 text-sm font-extrabold transition-all cursor-pointer whitespace-nowrap ${
              activeTab === 'operations'
                ? 'border-emerald-500 text-emerald-600'
                : 'border-transparent text-slate-400 hover:text-slate-600'
            }`}
          >
            <Sparkles className="h-4 w-4" />
            即時看板
          </button>
          <button
            id="tab_periodic"
            onClick={() => setActiveTab('periodic')}
            className={`flex items-center gap-2 border-b-2 px-6 py-3 text-sm font-extrabold transition-all cursor-pointer whitespace-nowrap ${
              activeTab === 'periodic'
                ? 'border-emerald-500 text-emerald-600'
                : 'border-transparent text-slate-400 hover:text-slate-600'
            }`}
          >
            <CalendarDays className="h-4 w-4" />
            週期統計
          </button>
          <button
            id="tab_analysis"
            onClick={() => setActiveTab('analysis')}
            className={`flex items-center gap-2 border-b-2 px-6 py-3 text-sm font-extrabold transition-all cursor-pointer whitespace-nowrap ${
              activeTab === 'analysis'
                ? 'border-emerald-500 text-emerald-600'
                : 'border-transparent text-slate-400 hover:text-slate-600'
            }`}
          >
            <Activity className="h-4 w-4" />
            耗用與採購分析
          </button>
          <button
            id="tab_history"
            onClick={() => setActiveTab('history')}
            className={`flex items-center gap-2 border-b-2 px-6 py-3 text-sm font-extrabold transition-all cursor-pointer whitespace-nowrap ${
              activeTab === 'history'
                ? 'border-emerald-500 text-emerald-600'
                : 'border-transparent text-slate-400 hover:text-slate-600'
            }`}
          >
            <History className="h-4 w-4" />
            歷史紀錄
          </button>
        </div>

        {activeTab === 'operations' ? (
          <>
            {/* Row 2: Quick entry Form (Left) & Data Breakdown Chart (Right) */}
            <section id="entry_form_and_guidance_row" className="grid grid-cols-1 gap-6 lg:grid-cols-2">
              {/* Quick Action Panel */}
              <div className="lg:col-span-1">
                <FruitForm
                  fruits={fruits}
                  inventory={inventory}
                  onLogSubmit={handleLogSubmit}
                />
              </div>

              {/* Data Breakdown Chart */}
              <div className="lg:col-span-1">
                <DataBreakdownChart
                  logs={logs}
                  fruits={fruits}
                />
              </div>
            </section>

            {/* Row 2.5: Live Inventory Cards (水果現況) */}
            <section id="inventory_cards_section">
              <InventorySummary
                fruits={fruits}
                inventory={inventory}
                totals={totals}
                onAddFruit={handleAddFruit}
                onUpdateFruitPrice={handleUpdateFruitPrice}
              />
            </section>

            {/* Row 3: Enlarged Recharts Analytics Charts */}
            <section id="enlarged_charts_section">
              <DashboardCharts
                logs={logs}
                fruits={fruits}
              />
            </section>
          </>
        ) : activeTab === 'periodic' ? (
          <section id="periodic_stats_section">
            <PeriodicStats
              logs={logs}
              fruits={fruits}
            />
          </section>
        ) : activeTab === 'analysis' ? (
          <section id="fruit_usage_analysis_tab_section">
            <FruitUsageAnalysis
              logs={logs}
              fruits={fruits}
            />
          </section>
        ) : (
          <div className="space-y-6">
            {/* Row 1: Detailed logs History Table */}
            <section id="history_table_section">
              <LogsTable
                logs={logs}
                fruits={fruits}
                onDeleteLog={handleDeleteLog}
                onExportCSV={handleExportCSV}
                onExportJSON={handleExportJSON}
                onImportJSON={handleImportJSON}
              />
            </section>
          </div>
        )}

      </main>

      {/* Elegant footer details */}
      <footer className="mt-16 border-t border-slate-100 bg-white py-8 text-center text-xs text-slate-400">
        <div className="mx-auto max-w-7xl px-4 sm:px-6 lg:px-8 space-y-2">
          <p>© 2026 飯店大廳迎賓水果統計與報廢防損系統. 版權所有.</p>
          <p className="text-[10px] font-mono text-slate-300">
            Hotel Lobby Fruit Tracker | Version 1.2 | System User ID: yazidane1009@gmail.com
          </p>
        </div>
      </footer>

    </div>
  );
}
