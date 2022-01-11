# Wordle Solver

This algorithm is designed to solve the game known as [Wordle](https://www.powerlanguage.co.uk/wordle/). It works by going through all the possible words and getting the frequency of each letter and the frequency of their positon in the word. This means that each word can be assigned a score based on the amount of common letters it has and where those letter are positioned allowing it to eliminate the most words possible with the least amount of tries.

## Frequency and Position Analysis


```python
with open("words.txt", "r") as f:
    words = []
    for i in f:
        words.append(i.rstrip())

letter_info = {}
for word in words:
    for idx, letter in enumerate(word):
        try:
            letter_info[letter]["freq"] += 1
        except:
            letter_info[letter] = {
                "freq": 1,
                "pos":{0:0,1:0,2:0,3:0,4:0}
            }
        letter_info[letter]["pos"][idx] += 1

word_scores = {}
for word in words:
    word_scores[word] = {
        "word": word,
        "freq_score": sum(set([letter_info[letter]["freq"] for letter in word])),
        "pos_score": sum([letter_info[letter]["pos"][idx] for idx, letter in enumerate(word)])
    }
    word_scores[word]["overall_score"] = word_scores[word]["freq_score"] + word_scores[word]["pos_score"]
```

# Storing Words in DB


```python
import sqlite3

conn = sqlite3.connect("words.db")
cur = conn.cursor()

cur.execute("""
CREATE TABLE IF NOT EXISTS words (
    word text PRIMARY KEY,
    let0 text NOT NULL,
    let1 text NOT NULL,
    let2 text NOT NULL,
    let3 text NOT NULL,
    let4 text NOT NULL,
    freq_score integer NOT NULL,
    pos_score integer NOT NULL,
    overall_score integer NOT NULL
);
""")

for word_data in word_scores:
    insert_tuple = (word_scores[word_data]["word"],) + tuple(word_scores[word_data]["word"]) + tuple(word_scores[word_data].values())[1:]
    cur.execute("""
    INSERT INTO words 
    (word, let0, let1, let2, let3, let4, freq_score, pos_score, overall_score) 
    VALUES (?,?,?,?,?,?,?,?,?)
    """,
    insert_tuple
    )
conn.commit()
```

# Actual Solving Part


```python
def suggestion(incorrect, correct, almost):
    incorrect = list(set(incorrect))
    correct = list(set(correct))
    almost = list(set(almost))
    
    query = "SELECT word, MAX(overall_score) FROM words WHERE "
    for let in incorrect:
        query += f"NOT word LIKE '%{let}%' AND "
    for let in almost:
        query += f"let{let[0]} <> '{let[1]}' AND word LIKE '%{let[1]}%' AND "
    for let in correct:
        query += f"let{let[0]} = '{let[1]}' AND "
    if query[-4:] == "AND ":
        query = query[:-4]
    else:
        query = query[:-6]
    cur.execute(query)

    optimal_word = cur.fetchone()
    return optimal_word

```

# Test it Yourself


```python
best_word = suggestion([],[],[])
print(f"Input '{best_word[0]}' as first word")
guesses = {
    "incorrect": [],
    "almost": [],
    "correct": []
}

for i in range(6):
    print("Input all the incorrect letters (e.g. rgb) - ")
    incorrect_letters = list(input(""))
    print("Input correct letter with wrong position (e.g. 4d 0e)- ")
    almost_letters = input().rsplit(" ")
    print("Input all correct letters (e.g. 2d 3c)")
    correct_letters = input().rsplit(" ")

    if correct_letters != ['']:
        correct_letters = [(int(x[0]), x[1]) for x in correct_letters]
    else: correct_letters = []
    if almost_letters != ['']:
        almost_letters = [(int(x[0]), x[1]) for x in almost_letters]
    else: almost_letters = []
    guesses["incorrect"] += incorrect_letters
    guesses["almost"] += almost_letters
    guesses["correct"] += correct_letters
    
    optimal_word = suggestion(guesses["incorrect"], guesses["correct"], guesses["almost"])
    print(f"\nMost optimal word is '{optimal_word[0]}'")
    
    print("Is {optimal_word[0]} correct y/N")
    finished = input().upper()
    if finished == "Y":
        print(f"Took {str(i+2)} attempts")
        break
```

# Performance Testing

- Average amount of guesses 4.502282641777417
- Max amount of guesses 7. 
- Min amount of guesses 1
- Guessed 9857 out of 10657 correctly with a success rate if 92.49%
- The standard deviation is 1.20


```python
import numpy as np

words = []
with open("words.txt", "r") as f:
    for i in f:
        words.append(i.rstrip())

correct_guesses = []
for answer in words:
    guesses = {
        "incorrect": [],
        "almost": [],
        "correct": []
    }
    for turn in range(7):
        optimal_word = suggestion(guesses["incorrect"], guesses["correct"], guesses["almost"])
        optimal_word = optimal_word[0]
        if optimal_word == answer:
            correct_guesses.append(turn+1)
            break
        for idx, let in enumerate(optimal_word):
            if let == answer[idx]:
                guesses["correct"].append((idx, let))
            elif let in list(answer):
                guesses["almost"].append((idx, optimal_word[idx]))
            else:
                guesses["incorrect"].append(let)

print(f"Average amount of guesses {sum(correct_guesses)/len(correct_guesses)}")
print(f"Max amount of guesses {max(correct_guesses)}. \nMin amount of guesses {min(correct_guesses)}")
print(f"Guessed {len(correct_guesses)} out of {len(words)} correctly with a success rate if {(len(correct_guesses)/len(words))*100:.2f}%")
print(f"The standard deviation is {np.std(correct_guesses):.2f}")
```

    Average amount of guesses 4.502282641777417
    Max amount of guesses 7. 
    Min amount of guesses 1
    Guessed 9857 out of 10657 correctly with a success rate if 92.49%
    The standard deviation is 1.20



