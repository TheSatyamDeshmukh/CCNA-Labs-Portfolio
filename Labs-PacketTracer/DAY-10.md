# Day 10 🔧 — Shell Scripting: Basics

> **"Ek script = hazaron commands ek baar mein!"**

---

## 🤔 Shell Script Kya Hota Hai?

Shell script ek **text file** hai jisme bahut saare commands likhe hote hain — ek baar chalao, sab commands ek ke baad ek apne aap chalte hain!

**Analogy:** Recipe card jaisi! Sabzi banane ki recipe mein: "pehle pyaz kaato, phir oil daalo, phir masala..." — ek ke baad ek steps. Script mein bhi aisa hi hota hai!

---

## 📄 Pehla Script — Step by Step

```bash
# Step 1: File banao
touch myfirst.sh

# Step 2: Edit karo
nano myfirst.sh
```

**File mein yeh likho:**
```bash
#!/bin/bash
# Yeh pehli line ZAROORI hai — "shebang" kehte hain isse
# Computer ko batata hai: "yeh bash script hai!"

echo "Hello Linux World!"
echo "Mera naam hai: $(whoami)"
echo "Aaj ki date hai: $(date)"
echo "Main hoon: $(pwd)"
```

```bash
# Step 3: Execute permission do
chmod +x myfirst.sh

# Step 4: Chalao!
./myfirst.sh
```

---

## 📦 Variables

```bash
#!/bin/bash

# Variable banana (= ke aas paas space BILKUL mat do!)
naam="Rahul"
umar=20
city="Delhi"

# Variable use karna — $ lagao!
echo "Mera naam $naam hai"
echo "Umar: $umar"
echo "Shahar: $city"

# Variable ko variable se
greeting="Hello $naam!"
echo $greeting

# Command ka output variable mein store karo
current_date=$(date)
current_user=$(whoami)
echo "Aaj: $current_date"
echo "User: $current_user"
```

**Special Variables:**
```bash
$0    # Script ka naam
$1    # Pehla argument
$2    # Doosra argument
$#    # Kitne arguments?
$@    # Sab arguments
$?    # Pichle command ka exit code (0 = success, non-0 = error)
$$    # Current script ka PID
```

---

## 💬 User Input Lena

```bash
#!/bin/bash

echo "Tumhara naam kya hai?"
read naam

echo "Tumhari umar kya hai?"
read umar

echo "Hello $naam! Tum $umar saal ke ho."

# Same line pe prompt:
read -p "Apna city batao: " city
echo "Tum $city mein rehte ho!"

# Hidden input (password ke liye):
read -s -p "Password: " password
echo ""
echo "Password save ho gaya!"
```

---

## 🔢 Math Operations

```bash
#!/bin/bash

a=10
b=3

sum=$((a + b))
diff=$((a - b))
product=$((a * b))
quotient=$((a / b))
remainder=$((a % b))

echo "Jod: $sum"
echo "Ghataav: $diff"
echo "Gunaa: $product"
echo "Bhaag: $quotient"
echo "Shesha: $remainder"
```

---

## ❓ If-Else Conditions

```bash
#!/bin/bash

marks=75

# Simple if
if [ $marks -ge 33 ]; then
    echo "Pass ho gaye!"
fi

# If-else
if [ $marks -ge 33 ]; then
    echo "Pass!"
else
    echo "Fail!"
fi

# If-elif-else
if [ $marks -ge 90 ]; then
    echo "Grade A — Excellent!"
elif [ $marks -ge 75 ]; then
    echo "Grade B — Good!"
elif [ $marks -ge 60 ]; then
    echo "Grade C — Average"
elif [ $marks -ge 33 ]; then
    echo "Grade D — Pass"
else
    echo "Fail!"
fi
```

**Comparison Operators:**
```bash
-eq    # Equal (==)
-ne    # Not equal (!=)
-gt    # Greater than (>)
-lt    # Less than (<)
-ge    # Greater than or equal (>=)
-le    # Less than or equal (<=)
```

**String Checks:**
```bash
[ "$a" == "$b" ]    # Equal strings
[ "$a" != "$b" ]    # Not equal
[ -z "$a" ]         # Empty string?
[ -n "$a" ]         # Not empty?
```

**File Checks:**
```bash
[ -f "file.txt" ]   # File exist karta hai?
[ -d "folder/" ]    # Directory exist karti hai?
[ -e "path" ]       # Exist karta hai (file ya folder)?
[ -r "file" ]       # Readable hai?
[ -w "file" ]       # Writable hai?
[ -x "file" ]       # Executable hai?
```

---

## 🔄 Loops

### For Loop:
```bash
#!/bin/bash

# Simple
for i in 1 2 3 4 5; do
    echo "Number: $i"
done

# Range
for i in {1..10}; do
    echo "Count: $i"
done

# Files ke saath
for file in *.txt; do
    echo "File mili: $file"
    wc -l "$file"
done

# C-style
for ((i=0; i<5; i++)); do
    echo "i = $i"
done
```

### While Loop:
```bash
#!/bin/bash

count=1
while [ $count -le 5 ]; do
    echo "Count: $count"
    count=$((count + 1))
done
```

---

## 🏋️ Practice — Aaj Ka Script

```bash
#!/bin/bash
# Grade Calculator

echo "================================"
echo "     Student Grade Calculator"
echo "================================"
read -p "Student ka naam: " naam
read -p "Marks (0-100): " marks

if [ $marks -ge 90 ]; then
    grade="A"
elif [ $marks -ge 75 ]; then
    grade="B"
elif [ $marks -ge 60 ]; then
    grade="C"
elif [ $marks -ge 33 ]; then
    grade="D"
else
    grade="F"
fi

echo ""
echo "Student: $naam"
echo "Marks: $marks"
echo "Grade: $grade"
echo "================================"
```

---

## 📝 Aaj Ka Summary

✅ `#!/bin/bash` — shebang, pehli line zaroori
✅ Variables — `naam="value"`, use karo `$naam` se
✅ `read` — user se input lo
✅ `$(( ))` — math operations
✅ `if-elif-else` — conditions
✅ `for` / `while` — loops
✅ `chmod +x` → `./script.sh` — chalao

---
**Day:** 10/30 | **Topic:** Shell Scripting Basics | **Difficulty:** ⭐⭐⭐☆☆
