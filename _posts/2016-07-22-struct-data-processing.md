---
title: Simple data processing with structs and functional methods
---

A question from [https://stackoverflow.com/questions/38530847](https://stackoverflow.com/questions/38530847), authored by [Josh Lopez](https://stackoverflow.com/users/4462954/josh-lopez)

> ### How to create a streak from an array of tuples in Swift
> 
> I have an array of tuples that looks like:
> 
>     [("07-20-2016", 2), ("07-22-2016", 5.0), ("07-21-2016", 3.3333333333333335)]
> 
> It consists of a String that is a date and a Double that is a mood (Moods less than or equal to 3 means that day was a good day).
> I am trying to figure out how to create a streak of good days.
> 
> So what I think I need to do first is sort the array of tuples to put them in order by date.
> Then I need to count the tuples that are below the mood value 3 if they are in a row. In this case the result would be 1 because there is only 1 day that is below 3.
> 
> If I had an array of tuples like:
> 
>     
>     [("07-22-2016", 5.0), ("07-21-2016", 3), ("07-20-2016", 2), ("07-19-2016", 1), ("07-18-2016", 1), ("07-16-2016", 2), ("07-15-2016", 3)]
> 
> It would have a result of 4 because the highest number of days in a row that are below 3 is 4. The reason it wouldn't be 6 is because a date of "07-17-2016" doesn't exist.

_(Edited to remove side commentary.)_
  
---

## [My reply](https://stackoverflow.com/a/38535661)

First, as matt suggested, make a nicer datatype. The language makes this super convenient with structs. We need each entry to have a date and a mood score. Easy-peasy.

    import Foundation

    struct JournalEntry {
        let date: NSDate
        let score: Double
    
        var isGoodDay: Bool { return self.score <= 3.0 }
    }

(The computed `isGoodDay` property will make life even easier later on.)

Now we need to turn the raw list of tuples into a list of these structs. That's straightforward. Create a date parser, and use `map` to convert between the tuple type and the struct:

    typealias RawEntry = (String, Double)
    func parseRawEntries<S: SequenceType where S.Generator.Element == RawEntry>
                        (rawEntries: S)
        -> [JournalEntry]
    {
        let parser = NSDateFormatter()
        parser.dateFormat = "MM-dd-yyyy"
        // Use guard and flatMap because dateFromString returns an Optional; 
        // you might prefer to throw an error to indicate that the data are 
        // incorrectly formatted
        return rawEntries.flatMap { rawEntry in
            guard let date = parser.dateFromString(rawEntry.0) else {
                return nil
            }
            return JournalEntry(date: date, score: rawEntry.1)
        }
    }

After converting we'll need to go through the list testing two conditions: first, that days are "good" and then that pairs of good days are consecutive. We have that helper method in the data structure itself to test the former. Let's create another helper function for the consecutive part:

    
    /* N.B. for brevity this assumes first and second are already sorted! */
    func entriesConsecutive(first: JournalEntry, _ second: JournalEntry) -> Bool {
        let calendar = NSCalendar.currentCalendar()
        let dayDiff = calendar.components(.Day,
                                          fromDate: first.date,
                                          toDate: second.date,
                                          options: NSCalendarOptions(rawValue: 0))
        return dayDiff.day == 1
    }

At each enumeration step, we'll have the current entry, but we also need either the previous or the next, so that we can test for consecutivity. We also obviously want to keep track of the streaks' counts as we find them. Let's make another datatype for that. (The reason for `lastEntry` rather than `nextEntry` will be clear in a moment.)

    struct Streak {
        let lastEntry: JournalEntry
        let count: UInt
    }

This is the fun part: iterating over the array, constructing `Streak`s. This can be done with a `for` loop. But consuming sequences by calculation and agglomeration on each new item is a known pattern. It's called "[folding][fold]"; Swift uses the alternate term `reduce()`.

Reducing a sequence into groups, or while keeping track of something, is also a well-known pattern: the combined value is itself a collection. At each step, you inspect the new item along with the most recent value in the reduction. The new value for the reduction -- what you return from the combining function -- is still the collection, either with the previous value updated or a new value appended.

The `Streak` datatype from above will make the elements in the reduction: that's why it uses the _last_ entry. Each step, we will save the `JournalEntry` and inspect it on the next.

For streaks, we only care about "good" days. Let's make life simple and `filter` the list right off the bat: `let goodDays = journal.filter { $0.isGoodDay }` We'll need the first entry so that there's something to compare to on the first step. This is also a chance to bail out if for some reason there _are_ no good days.

    guard let firstEntry = goodDays.first else {
        return []
    }
    
That first entry goes into a `Streak` object, which goes into the collection that will become the reduction: `let initial = [Streak(lastEntry: firstEntry, count: 1)]`, and we're ready to go. The full function looks like this (I'll let the comments explain the procedure inside the reduce):

    func findStreaks(in journal: [JournalEntry]) -> [Streak] {
        let goodDays = journal.filter { $0.isGoodDay }
        guard let firstEntry = goodDays.first else {
            return []
        }
        let initial = [Streak(lastEntry: firstEntry, count: 1)]
        return 
            // The first entry is already used; skip it.
            goodDays.dropFirst()
                    .reduce(initial) { 
                      (reduction: [Streak], entry: JournalEntry) in
                        // Get most recent Streak for inspection
                        let streak = reduction.last!    // ! We know this exists
                
                        let consecutive = entriesConsecutive(streak.lastEntry, 
                                                             entry)
                        // If this is the same streak, increment the count by
                        // replacing the previous value.
                        // Otherwise add a new Streak with count 1.
                        let newCount = consecutive ? streak.count + 1 : 1
                        let streaks = consecutive ? 
                                      Array(reduction.dropLast()) :
                                      reduction
                        return streaks + 
                                [Streak(lastEntry: entry, count: newCount)]
                    }
    }
    
Now, back at the top level, things are very simple. Raw data:

    let rawEntries = [("07-22-2016", 5.0), ("07-21-2016", 3), ("07-20-2016", 2), ("07-19-2016", 1), ("07-18-2016", 1), ("07-16-2016", 2), ("07-15-2016", 3)]
    
Convert that into the relevant datatype, immediately sorting by date while we're at it.

    let journal = parseRawEntries(rawEntries).sort { $0.date < $1.date }
    
Calculate streaks:

    let streaks = findStreaks(in: journal)

And the result you want is then `let bestStreakLen = streaks.map({ $0.count }).maxElement()`
    
---

Small note: comparing `NSDate` using `<` like that requires a new function:

    func <(lhs: NSDate, rhs: NSDate) -> Bool {
        let earlier = lhs.earlierDate(rhs)
        return earlier == lhs
    }
    
[fold]:https://en.wikipedia.org/wiki/Fold_(higher-order_function)
