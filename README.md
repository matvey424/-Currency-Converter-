import tkinter as tk
from tkinter import ttk, messagebox, scrolledtext

import json
import os
from datetime import datetime
from typing import Dict, Optional

class CurrencyConverter:
    """Приложение для конвертации валют с использованием внешнего API"""
    
    def __init__(self, root):
        self.root = root
        self.root.title("Currency Converter")
        self.root.geometry("900x700")
        self.root.minsize(800, 600)
        
        # Конфигурация
        self.api_key = self.load_api_key()
        self.base_url = "https://v6.exchangerate-api.com/v6"
        self.history_file = "conversion_history.json"
        self.rates_cache_file = "rates_cache.json"
        
        # Данные
        self.exchange_rates = {}
        self.history = []
        self.currencies = []
        
        # Загрузка данных
        self.load_history()
        self.load_cached_rates()
        self.setup_ui()
        
        # Загрузка курсов при старте
        if self.api_key:
            self.update_rates()
        else:
            self.show_api_key_dialog()

    def setup_ui(self):
        """Настройка пользовательского интерфейса"""
        # Главный контейнер
        main_frame = ttk.Frame(self.root, padding="10")
        main_frame.grid(row=0, column=0, sticky=(tk.W, tk.E, tk.N, tk.S))
        
        self.root.columnconfigure(0, weight=1)
        self.root.rowconfigure(0, weight=1)
        main_frame.columnconfigure(0, weight=1)
        main_frame.columnconfigure(1, weight=1)
        main_frame.rowconfigure(2, weight=1)
        
        # Левая панель - Конвертер
        left_frame = ttk.LabelFrame(main_frame, text="Конвертер валют", padding="10")
        left_frame.grid(row=0, column=0, rowspan=3, sticky=(tk.W, tk.E, tk.N, tk.S), padx=(0, 5))
        left_frame.columnconfigure(1, weight=1)
        
        # API Key статус
        self.api_status_label = ttk.Label(left_frame, text="API Key: не настроен", foreground="red")
        self.api_status_label.grid(row=0, column=0, columnspan=2, sticky=tk.W, pady=(0, 10))
        
        ttk.Button(left_frame, text="Настроить API Key", command=self.show_api_key_dialog).grid(
            row=0, column=2, sticky=tk.E, pady=(0, 10))
        
        # Выбор валюты "Из"
        ttk.Label(left_frame, text="Из валюты:").grid(row=1, column=0, sticky=tk.W, pady=(0, 5))
        self.from_currency = ttk.Combobox(left_frame, values=self.currencies, state="readonly", width=10)
        self.from_currency.grid(row=1, column=1, sticky=(tk.W, tk.E), pady=(0, 5), padx=(5, 0))
        if self.currencies:
            self.from_currency.set("USD")
        
        # Выбор валюты "В"
        ttk.Label(left_frame, text="В валюту:").grid(row=2, column=0, sticky=tk.W, pady=(0, 5))
        self.to_currency = ttk.Combobox(left_frame, values=self.currencies, state="readonly", width=10)
        self.to_currency.grid(row=2, column=1, sticky=(tk.W, tk.E), pady=(0, 5), padx=(5, 0))
        if len(self.currencies) > 1:
            self.to_currency.set("EUR")
        
        # Поле ввода суммы
        ttk.Label(left_frame, text="Сумма:").grid(row=3, column=0, sticky=tk.W, pady=(0, 5))
        self.amount_entry = ttk.Entry(left_frame, font=('Arial', 12))
        self.amount_entry.grid(row=3, column=1, columnspan=2, sticky=(tk.W, tk.E), pady=(0, 5), padx=(5, 0))
        self.amount_entry.insert(0, "1.00")
        
        # Кнопка конвертации
        self.convert_button = ttk.Button(left_frame, text="Конвертировать", command=self.convert)
        self.convert_button.grid(row=4, column=1, pady=(10, 10), sticky=tk.E)
        
        # Результат конвертации
        ttk.Label(left_frame, text="Результат:").grid(row=5, column=0, sticky=tk.W, pady=(0, 5))
        self.result_label = ttk.Label(left_frame, text="0.00", font=('Arial', 16, 'bold'))
        self.result_label.grid(row=5, column=1, columnspan=2, sticky=tk.W, pady=(0, 5), padx=(5, 0))
        
        # Информация о курсе
        self.rate_info_label = ttk.Label(left_frame, text="Актуальный курс будет показан здесь", font=('Arial', 9))
        self.rate_info_label.grid(row=6, column=0, columnspan=3, sticky=tk.W, pady=(0, 10))
        
        # Кнопка обновления курсов
        ttk.Button(left_frame, text="Обновить курсы валют", command=self.update_rates).grid(
            row=7, column=0, columnspan=3, pady=(0, 10))
        
        # Правая панель - История
        right_frame = ttk.LabelFrame(main_frame, text="История конвертаций", padding="10")
        right_frame.grid(row=0, column=1, rowspan=3, sticky=(tk.W, tk.E, tk.N, tk.S))
        right_frame.columnconfigure(0, weight=1)
        right_frame.rowconfigure(1, weight=1)
        
        # Кнопки управления историей
        history_buttons = ttk.Frame(right_frame)
        history_buttons.grid(row=0, column=0, sticky=(tk.W, tk.E), pady=(0, 5))
        
        ttk.Button(history_buttons, text="Очистить историю", command=self.clear_history).pack(side=tk.LEFT, padx=(0, 5))
        ttk.Button(history_buttons, text="Экспорт в CSV", command=self.export_history).pack(side=tk.LEFT)
        
        # Таблица истории
        self.history_tree = ttk.Treeview(right_frame, columns=('from', 'to', 'amount', 'result', 'rate', 'date'), 
                                         show='headings', height=15)
        
        self.history_tree.heading('from', text='Из')
        self.history_tree.heading('to', text='В')
        self.history_tree.heading('amount', text='Сумма')
        self.history_tree.heading('result', text='Результат')
        self.history_tree.heading('rate', text='Курс')
        self.history_tree.heading('date', text='Дата')
        
        self.history_tree.column('from', width=80)
        self.history_tree.column('to', width=80)
        self.history_tree.column('amount', width=100)
        self.history_tree.column('result', width=100)
        self.history_tree.column('rate', width=100)
        self.history_tree.column('date', width=150)
        
        self.history_tree.grid(row=1, column=0, sticky=(tk.W, tk.E, tk.N, tk.S))
        
        # Скроллбар для таблицы
        scrollbar = ttk.Scrollbar(right_frame, orient=tk.VERTICAL, command=self.history_tree.yview)
        scrollbar.grid(row=1, column=1, sticky=(tk.N, tk.S))
        self.history_tree.configure(yscrollcommand=scrollbar.set)
        
        # Привязка событий
        self.amount_entry.bind('<Return>', lambda e: self.convert())
        self.root.bind('<Control-r>', lambda e: self.update_rates())
        
        # Обновление истории
        self.refresh_history_table()
        
    def show_api_key_dialog(self):
        """Диалог ввода API ключа"""
        dialog = tk.Toplevel(self.root)
        dialog.title("Настройка API Key")
        dialog.geometry("400x200")
        dialog.transient(self.root)
        dialog.grab_set()
        
        ttk.Label(dialog, text="Введите ваш API ключ от exchangerate-api.com:", 
                 font=('Arial', 10)).pack(pady=10, padx=20)
        
        ttk.Label(dialog, text="Получить ключ: https://www.exchangerate-api.com", 
                 foreground="blue").pack(pady=5)
        
        api_entry = ttk.Entry(dialog, width=40, show="*")
        api_entry.pack(pady=10, padx=20)
        if self.api_key:
            api_entry.insert(0, self.api_key)
        
        def save_key():
            key = api_entry.get().strip()
            if key:
                self.api_key = key
                self.save_api_key(key)
                self.api_status_label.config(text=f"API Key: {key[:10]}...", foreground="green")
                self.update_rates()
                dialog.destroy()
            else:
                messagebox.showwarning("Предупреждение", "API ключ не может быть пустым")
        
        ttk.Button(dialog, text="Сохранить", command=save_key).pack(pady=10)
        
    def save_api_key(self, key):
        """Сохранение API ключа в файл"""
        try:
            with open('api_key.json', 'w') as f:
                json.dump({'api_key': key}, f)
        except IOError as e:
            messagebox.showerror("Ошибка", f"Не удалось сохранить API ключ: {e}")
    
    def load_api_key(self) -> Optional[str]:
        """Загрузка API ключа из файла"""
        try:
            if os.path.exists('api_key.json'):
                with open('api_key.json', 'r') as f:
                    data = json.load(f)
                    return data.get('api_key')
        except (json.JSONDecodeError, IOError):
            pass
        return None
    
    def update_rates(self):
        """Обновление курсов валют через API"""
        if not self.api_key:
            messagebox.showwarning("Предупреждение", "Сначала настройте API ключ")
            return
        
        try:
            url = f"{self.base_url}/{self.api_key}/latest/USD"
            response = requests.get(url, timeout=10)
            response.raise_for_status()
            
            data = response.json()
            
            if data.get('result') == 'success':
                self.exchange_rates = data.get('conversion_rates', {})
                self.currencies = sorted(list(self.exchange_rates.keys()))
                
                # Обновление выпадающих списков
                self.from_currency['values'] = self.currencies
                self.to_currency['values'] = self.currencies
                
                # Сохранение кэша курсов
                self.save_cached_rates()
                
                messagebox.showinfo("Успех", "Курсы валют обновлены!")
                self.api_status_label.config(text="API Key: активен", foreground="green")
            else:
                messagebox.showerror("Ошибка", f"Ошибка API: {data.get('error-type', 'Неизвестная ошибка')}")
                
        except requests.exceptions.RequestException as e:
            messagebox.showerror("Ошибка", f"Не удалось получить курсы валют: {e}")
            # Попытка загрузить кэшированные курсы
            if not self.exchange_rates:
                self.load_cached_rates()
    
    def save_cached_rates(self):
        """Сохранение кэшированных курсов"""
        try:
            with open(self.rates_cache_file, 'w') as f:
                json.dump({
                    'rates': self.exchange_rates,
                    'currencies': self.currencies,
                    'timestamp': datetime.now().isoformat()
                }, f, indent=2)
        except IOError:
            pass
    
    def load_cached_rates(self):
        """Загрузка кэшированных курсов"""
        try:
            if os.path.exists(self.rates_cache_file):
                with open(self.rates_cache_file, 'r') as f:
                    data = json.load(f)
                    self.exchange_rates = data.get('rates', {})
                    self.currencies = data.get('currencies', [])
        except (json.JSONDecodeError, IOError):
            pass
    
    def convert(self):
        """Выполнение конвертации валют"""
        if not self.exchange_rates:
            messagebox.showwarning("Предупреждение", "Сначала обновите курсы валют")
            return
        
        # Получение введенных данных
        from_curr = self.from_currency.get()
        to_curr = self.to_currency.get()
        amount_str = self.amount_entry.get().strip()
        
        # Валидация
        if not from_curr or not to_curr:
            messagebox.showwarning("Предупреждение", "Выберите валюты")
            return
        
        if from_curr == to_curr:
            messagebox.showwarning("Предупреждение", "Выберите разные валюты")
            return
        
        try:
            amount = float(amount_str)
            if amount <= 0:
                messagebox.showwarning("Предупреждение", "Сумма должна быть положительным числом")
                return
        except ValueError:
            messagebox.showwarning("Предупреждение", "Введите корректное числовое значение")
            return
        
        # Конвертация
        if from_curr in self.exchange_rates and to_curr in self.exchange_rates:
            # Конвертация через USD (базовая валюта API)
            from_rate = self.exchange_rates[from_curr]
            to_rate = self.exchange_rates[to_curr]
            
            # Конвертация: сначала в USD, потом в целевую валюту
            amount_in_usd = amount / from_rate
            result = amount_in_usd * to_rate
            exchange_rate = to_rate / from_rate
            
            # Отображение результата
            self.result_label.config(text=f"{result:.2f} {to_curr}")
            self.rate_info_label.config(text=f"Курс: 1 {from_curr} = {exchange_rate:.4f} {to_curr}")
            
            # Сохранение в историю
            self.add_to_history(from_curr, to_curr, amount, result, exchange_rate)
            self.refresh_history_table()
            
        else:
            messagebox.showerror("Ошибка", "Выбранные валюты не найдены в курсах")
    
    def add_to_history(self, from_curr, to_curr, amount, result, rate):
        """Добавление записи в историю конвертаций"""
        record = {
            'from': from_curr,
            'to': to_curr,
            'amount': amount,
            'result': result,
            'rate': rate,
            'date': datetime.now().strftime('%Y-%m-%d %H:%M:%S')
        }
        self.history.append(record)
        self.save_history()
    
    def refresh_history_table(self):
        """Обновление таблицы истории"""
        # Очистка таблицы
        for item in self.history_tree.get_children():
            self.history_tree.delete(item)
        
        # Заполнение таблицы (последние 50 записей)
        for record in self.history[-50:]:
            self.history_tree.insert('', 0, values=(
                record['from'],
                record['to'],
                f"{record['amount']:.2f}",
                f"{record['result']:.2f}",
                f"{record['rate']:.4f}",
                record['date']
            ))
    
    def save_history(self):
        """Сохранение истории в файл"""
        try:
            with open(self.history_file, 'w', encoding='utf-8') as f:
                json.dump(self.history, f, ensure_ascii=False, indent=2)
        except IOError as e:
            messagebox.showerror("Ошибка", f"Не удалось сохранить историю: {e}")
    
    def load_history(self):
        """Загрузка истории из файла"""
        try:
            if os.path.exists(self.history_file):
                with open(self.history_file, 'r', encoding='utf-8') as f:
                    self.history = json.load(f)
        except (json.JSONDecodeError, IOError):
            self.history = []
    
    def clear_history(self):
        """Очистка истории конвертаций"""
        result = messagebox.askyesno("Подтверждение", "Вы уверены, что хотите очистить всю историю?")
        if result:
            self.history = []
            self.save_history()
            self.refresh_history_table()
    
    def export_history(self):
        """Экспорт истории в CSV"""
        try:
            import csv
            with open('conversion_history.csv', 'w', newline='', encoding='utf-8') as f:
                writer = csv.writer(f)
                writer.writerow(['Дата', 'Из валюты', 'В валюту', 'Сумма', 'Результат', 'Курс'])
                for record in self.history:
                    writer.writerow([
                        record['date'],
                        record['from'],
                        record['to'],
                        record['amount'],
                        record['result'],
                        record['rate']
                    ])
            messagebox.showinfo("Успех", "История экспортирована в conversion_history.csv")
        except Exception as e:
            messagebox.showerror("Ошибка", f"Не удалось экспортировать историю: {e}")

if __name__ == "__main__":
    root = tk.Tk()
    app = CurrencyConverter(root)
    root.mainloop()
