# reto_07

Este código implementa un sistema de gestión de pedidos en un restaurante, con un menú dinámico que permite agregar, actualizar y eliminar elementos almacenados en un archivo JSON mediante la interfaz `MenuManager`. Se utiliza `namedtuple` (`MenuItemTuple`) para definir los elementos del menú de manera eficiente. Además, se incorporó una cola FIFO (`OrderQueue`) para manejar múltiples pedidos en orden de llegada. La clase `Order` permite a los clientes agregar elementos a su pedido y calcular el total con descuentos aplicados según el monto y la composición del pedido. 
Para facilitar la visualización de los cambios y acelerar el proceso de prueba, se eliminaron algunas funcionalidades del código original del reto 04, como la solicitud de características adicionales de los alimentos (calorías, si es vegetariano, tiempo de cocción, etc.), permitiendo verificar más rápidamente el correcto funcionamiento de las nuevas funciones añadidas.

``` python
import json
from collections import namedtuple, deque
from abc import ABC, abstractmethod

# Named tuple for defining a menu item
MenuItemTuple = namedtuple("MenuItemTuple", ["name", "price"])

class MenuItem:
    def __init__(self, name: str, price: float) -> None:
        self.__name = name
        self.__price = price
    
    def get_name(self):
        return self.__name
    
    def set_name(self, name):
        self.__name = name
    
    def get_price(self):
        return self.__price
    
    def set_price(self, price):
        self.__price = price
    
    def __str__(self):
        return f"{self.__name} - ${self.__price}"

class MenuManager(ABC):
    """ Interface for managing the menu """
    @abstractmethod
    def add_item(self, name: str, price: float):
        pass
    
    @abstractmethod
    def update_item(self, name: str, price: float):
        pass
    
    @abstractmethod
    def delete_item(self, name: str):
        pass
    
    @abstractmethod
    def save_to_json(self, filename: str):
        pass
    
    @abstractmethod
    def load_from_json(self, filename: str):
        pass

class Menu(MenuManager):
    def __init__(self) -> None:
        self.items = []
        self.load_from_json("menu.json")
    
    def add_item(self, name: str, price: float):
        self.items.append(MenuItem(name, price))
    
    def update_item(self, name: str, price: float):
        for item in self.items:
            if item.get_name().lower() == name.lower():
                item.set_price(price)
                return
    
    def delete_item(self, name: str):
        self.items = [item for item in self.items if item.get_name().lower() != name.lower()]
    
    def save_to_json(self, filename: str):
        with open(filename, "w") as file:
            json.dump([{ "name": item.get_name(), "price": item.get_price() } for item in self.items], file)
    
    def load_from_json(self, filename: str):
        try:
            with open(filename, "r") as file:
                data = json.load(file)
                self.items = [MenuItem(name, price) for name, price in data.items()]
        except (FileNotFoundError, json.JSONDecodeError):
            self.items = []
    
    def get_item(self, name: str):
        for item in self.items:
            if item.get_name().lower() == name.lower():
                return item
        return None

# FIFO Queue for handling orders
class OrderQueue:
    def __init__(self):
        self.queue = deque()
    
    def add_order(self, order):
        self.queue.append(order)
    
    def process_order(self):
        if self.queue:
            return self.queue.popleft()
        else:
            print("No orders to process.")
            return None
    
class Order:
    def __init__(self, menu: Menu) -> None:
        self.menu = menu
        self.menu_items = []

    def add_item(self, name: str):
        item = self.menu.get_item(name)
        if item:
            self.menu_items.append(item)
        else:
            print(f"Item '{name}' is not in the menu.")

    def apply_discount(self) -> float:
        total_price = sum(item.get_price() for item in self.menu_items)
        discount = 0

        if total_price >= 50:
            discount = 0.2
        elif total_price >= 30:
            discount = 0.1

        return discount

    def calculate_total_price(self) -> float:
        total_price = sum(item.get_price() for item in self.menu_items)
        discount = self.apply_discount()
        return total_price * (1 - discount)


class Payment(ABC):
    @abstractmethod
    def pay(self, total_price: float) -> bool:
        pass

class CreditCard(Payment):
    def __init__(self, number: str, cvv: str, balance: float) -> None:
        self.__number = number
        self.__cvv = cvv
        self.__balance = balance
    
    def pay(self, total_price: float) -> bool:
        if self.__balance >= total_price:
            self.__balance -= total_price
            print(f"Payment of ${total_price} successful with card {self.__number[-4:]}")
            return True
        else:
            print("Insufficient balance on the credit card.")
            return False

class Cash(Payment):
    def __init__(self, amount: float) -> None:
        self.__amount = amount
    
    def pay(self, total_price: float) -> bool:
        if self.__amount >= total_price:
            self.__amount -= total_price
            print(f"Payment of ${total_price} successful with cash.")
            return True
        else:
            print(f"Insufficient cash. Need ${total_price - self.__amount}.")
            return False

if __name__ == "__main__":
    # Example usage
    menu = Menu()
    order_queue = OrderQueue()
    order1 = Order(menu)
    order2 = Order(menu)

    my_cash = Cash(60)
    my_credit_card = CreditCard("1234567890123456", "123", 500)

    order1.add_item("Pizza")
    order1.add_item("Coffee")
    order1.add_item("Spaghetti")
    order1.add_item("Burger")
    order1.add_item("Salad")
    order_queue.add_order(order1)
    order2.add_item("Salad")
    order2.add_item("Burger")
    order_queue.add_order(order2)


    while order_queue.queue:  
        processed_order = order_queue.process_order()
        if processed_order:
            total_price = processed_order.calculate_total_price()
            print(f"Total price: ${total_price}")
            
            choice = input("Choose your payment method (cash/credit): ").strip().lower()
            if choice == "cash":
                my_cash.pay(total_price)
            elif choice == "credit":
                my_credit_card.pay(total_price)
            else:
                print("Invalid payment method.")

```
