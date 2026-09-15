# TryHackMe: Python Basics Walkthrough
This walkthrough documents the step-by-step execution of code exercises, tasks, and flag acquisitions for the Python Basics room on TryHackMe.

### Task 2: Hello World
Concept: Introduction to basic string output using ```print()``` and line comments (```#```).
Code Executed (```script.py```)
```python
print('Hello World');
```
- Output: ```Hello World```
- Flag: ```THM{PRINT_STATEMENTS}```

### Task 3: Mathematical Operators
Concept: Working with basic arithmetic operations in Python (+, -, *, ).
1. Addition
- Code:
```python
a = 21
b = 43
c = a + b
print(c)
```
- Output: ```64```
- Flag: ```THM{ADDITION}```

2. Subtraction
- Code:
```python
a = 142
b = 52
c = a - b
print(c)
```
- Output: ```90```
- Flag: ```THM{SUBTRCT}```

3. Multiplication
- Code:
```python
a = 10
b = 342
c = a * b
print(c)
```
- Output: ```3420```
- Flag: ```THM{MULTIPLICATION_PYTHON}```

4. Exponentiation (Power)
- Code:
```python
a = 5
print(a**2)
```
- Output: ```25```
- Flag: ```THM{EXPON3NT_POWER}```

###  Task 4: Variables and Data Types
Concept: Declaring variables, reassigning values, and performing dynamic variable updates.
Code Executed (```script.py```)
```python
height = 200
height = height + 50
print(height)
```
- Output: ```250```
- Flag: ```THM{VARIABL3S}```

### Task 6: Introduction to If Statements (Shipping Project)
Concept: Implementing conditional decision-making logic (```if / else```) to dynamically calculate shipping costs based on basket value.
Initial Implementation (```shipping.py```)
```python
X = 100
shipping_cost_per_kg = 1.20
customer_basket_cost = 34
customer_basket_weight = 44

if(customer_basket_cost >= X):
    print('Free shipping!')
else:
    shipping_cost = customer_basket_weight * shipping_cost_per_kg
    customer_basket_cost = customer_basket_cost + shipping_cost

print("Total basket cost including shipping is " + str(customer_basket_cost))
```
- Result (Cost = 34):
  - Output: ```Total basket cost including shipping is 86.8```
  - Flag: ```THM{IF_STATEMENT_SHOPPING}```
- Updated Condition (Cost = 101):
  - Update line 15 to: ```customer_basket_cost = 101```
  - Output:
```plaintext
Free shipping!
Total basket cost including shipping is 101
```
- Flag: ```THM{MY_FIRST_APP}```

### Task 7: Loops
Concept: Iterating through numerical sequences using the ```range()``` function and for loops.
Code Executed (```script.py```)
```python
for i in range(50 + 1):
    print(i)
```
- Output: ```Numbers 0 through 50 printed sequentially```
- Flag: ```THM{L0OPS_WHILE_FOR}```

### Task 8: Introduction to Functions (Bitcoin Project)
Concept: Defining reusable modular code blocks using ```def```, passing arguments, and returning calculated values.
Code Executed (```bitcoin.py```)
```python
investment_in_bitcoin = 1.2
bitcoin_to_usd = 40000

# Function definition
def bitcoinToUSD(bitcoin_amount, bitcoin_value_usd):
    usd_value = bitcoin_amount * bitcoin_value_usd
    return usd_value

investment_in_usd = bitcoinToUSD(investment_in_bitcoin, bitcoin_to_usd)

if investment_in_usd <= 30000:
    print("Investment below $30,000! SELL!")
else:
    print("Investment above $30,000")
```
- Output: Investment above $30,000

    Flag: THM{BITCOIN_INVESTOR}

### Task 9: File Handling
Concept: Interacting with local files using Python's built-in ```open()```, ```.read()```, and ```.close()``` methods.
Code Executed (```script.py```)
```python
f = open("flag.txt", "r")
print(f.read())
```
Summary Matrix
|Task      |Topic            |Core Concept / File              |Flag                                                                            |
|:--       |:--              |:--                              |:--                                                                             |
|Task 2    |Hello World      |Output & Comments                |THM{PRINT_STATEMENTS}                                                           |
|Task 3    |Math Operators   |Arithmetic Operations            |"THM{ADDITION}, THM{SUBTRCT}, THM{MULTIPLICATION_PYTHON}, THM{EXPON3NT_POWER}"  |
|Task 4    |Variables        |Variable Reassignment            |THM{VARIABL3S}                                                                  |
|Task 6    |If Statements    |Conditionals (shipping.py)       |"THM{IF_STATEMENT_SHOPPING}, THM{MY_FIRST_APP}"                                 |
|Task 7    |Loops            |for i in range()                 |THM{L0OPS_WHILE_FOR}                                                            |
|Task 8    |Functions        |def & return (bitcoin.py)        |THM{BITCOIN_INVESTOR}                                                           | 
|Task 9    |Files            |open() & .read()                 |THM{F1LE_R3AD}                                                                  |
