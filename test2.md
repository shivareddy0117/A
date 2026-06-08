#test2


# Coding Round Solution: Character Indexer

## Problem Statement

We are given a list of books. Each book contains text lines.

We need to build a system that reads all books line by line and creates an index of characters with the number of times each character appears.

### Example

Harry Potter books have characters like:

- Harry
- Hermione
- Ron

Lord of the Rings books have characters like:

- Gandalf
- Bilbo
- Gollum

If Harry appears 500 times and Hermione appears 400 times, return:

```text
Harry -> 500
Hermione -> 400
```

The design should be extensible, so later we can add more character names without changing the main counting logic.

---

## Input

```java
List<Book> books = Arrays.asList(
    new Book("Harry Potter", Arrays.asList(
        "Harry went with Hermione.",
        "Ron saw Harry.",
        "Hermione helped Harry."
    )),
    new Book("Lord of the Rings", Arrays.asList(
        "Gandalf met Bilbo.",
        "Gollum followed Bilbo."
    ))
);

Set<String> charactersToIndex = new HashSet<>(
    Arrays.asList("Harry", "Hermione", "Ron", "Gandalf", "Bilbo", "Gollum")
);
```

## Expected Output

```text
Harry -> 3
Hermione -> 2
Ron -> 1
Gandalf -> 1
Bilbo -> 2
Gollum -> 1
```

---

## Approach

1. Create a `Book` class with book name and lines.

2. Create a `BookParser` interface because the question says there is already an API like:

   ```java
   List<String> getLines(Book book)
   ```

3. Create a `CharacterIndexer` class.

4. Store all characters we want to count inside a `Set<String>`.

5. For every book:

   - Get all lines using `bookParser.getLines(book)`.
   - For each line, split it into words.
   - Clean punctuation like `.`, `,`, `!`, and `?`.
   - Check if the word exists in our character set.
   - If yes, increase the count in a `Map<String, Integer>`.

6. Return the map containing character counts.

The main advantage of this design is extensibility. Tomorrow, if we want to count `Dumbledore`, `Frodo`, or `Voldemort`, we only add them to the character set. We do not need to change the indexing logic.

---

## Java Code

```java
import java.util.*;

class Book {
    private String name;
    private List<String> lines;

    public Book(String name, List<String> lines) {
        this.name = name;
        this.lines = lines;
    }

    public String getName() {
        return name;
    }

    public List<String> getLines() {
        return lines;
    }
}

interface BookParser {
    List<String> getLines(Book book);
}

class SimpleBookParser implements BookParser {
    @Override
    public List<String> getLines(Book book) {
        return book.getLines();
    }
}

class CharacterIndexer {
    private BookParser bookParser;
    private Set<String> charactersToIndex;

    public CharacterIndexer(BookParser bookParser, Set<String> charactersToIndex) {
        this.bookParser = bookParser;
        this.charactersToIndex = charactersToIndex;
    }

    public Map<String, Integer> buildIndex(List<Book> books) {
        Map<String, Integer> characterCountMap = new HashMap<>();

        for (Book book : books) {
            List<String> lines = bookParser.getLines(book);

            for (String line : lines) {
                countCharactersInLine(line, characterCountMap);
            }
        }

        return characterCountMap;
    }

    private void countCharactersInLine(String line, Map<String, Integer> characterCountMap) {
        String[] words = line.split("\\s+");

        for (String word : words) {
            String cleanedWord = cleanWord(word);

            if (charactersToIndex.contains(cleanedWord)) {
                characterCountMap.put(
                    cleanedWord,
                    characterCountMap.getOrDefault(cleanedWord, 0) + 1
                );
            }
        }
    }

    private String cleanWord(String word) {
        return word.replaceAll("[^a-zA-Z]", "");
    }
}

public class Main {
    public static void main(String[] args) {
        List<Book> books = Arrays.asList(
            new Book("Harry Potter", Arrays.asList(
                "Harry went with Hermione.",
                "Ron saw Harry.",
                "Hermione helped Harry."
            )),
            new Book("Lord of the Rings", Arrays.asList(
                "Gandalf met Bilbo.",
                "Gollum followed Bilbo."
            ))
        );

        Set<String> charactersToIndex = new HashSet<>(
            Arrays.asList("Harry", "Hermione", "Ron", "Gandalf", "Bilbo", "Gollum")
        );

        BookParser parser = new SimpleBookParser();
        CharacterIndexer indexer = new CharacterIndexer(parser, charactersToIndex);

        Map<String, Integer> result = indexer.buildIndex(books);

        for (Map.Entry<String, Integer> entry : result.entrySet()) {
            System.out.println(entry.getKey() + " -> " + entry.getValue());
        }
    }
}
```

---

## Output

```text
Harry -> 3
Hermione -> 2
Ron -> 1
Gandalf -> 1
Bilbo -> 2
Gollum -> 1
```

> Note: The order may vary because `HashMap` does not guarantee insertion order.

---

## Edge Cases

### 1. Empty Book List

**Input:**

```java
[]
```

**Output:**

```java
{}
```

---

### 2. Book Has Empty Lines

**Input:**

```java
["", "   "]
```

**Output:**

```java
{}
```

---

### 3. Character Appears With Punctuation

**Input:**

```text
Harry, Harry. Harry!
```

**Output:**

```text
Harry -> 3
```

---

### 4. Character Is Not in the Configured Character List

**Input:**

```text
Dumbledore helped Harry
```

**Characters to index:**

```text
Harry
```

**Output:**

```text
Harry -> 1
```

---

### 5. Duplicate Character Names in Input List

Using a `Set<String>` automatically removes duplicate character names.

---

## Time Complexity

Let:

- `B` = number of books
- `L` = total number of lines across all books
- `W` = total number of words across all lines

We process every word once.

Lookup in a `HashSet` usually takes:

```text
O(1)
```

Therefore, the total time complexity is:

```text
O(W)
```

---

## Space Complexity

Let:

- `C` = number of characters we are indexing

We store:

- `Set<String> charactersToIndex`
- `Map<String, Integer> characterCountMap`

Therefore, the space complexity is:

```text
O(C)
```

Space depends mainly on the number of character names, not on the number of books.
