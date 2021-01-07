---
title: haskell
layout: markmap
date: 2020-12-23 17:07:53
---

# Haskell
## types
### defination `data BookInfo = Book Int String [String]`
#### type constructor `BookInfo`
#### value constructor `Book`
#### components `Int` `String` `[String]`
### type synonyms `type CustomerID = Int`
### algebraic data types `data Bool = False | True`
### Pattern matching
```Haskell
sumList (x:xs) = x + sumList xs
sumList []     = 0
```
### Record syntax
```Haskell
data CalendarTime = CalendarTime {
  ctYear                      :: Int,
  ctMonth                     :: Month,
  ctDay, ctHour, ctMin, ctSec :: Int,
  ctPicosec                   :: Integer,
  ctWDay                      :: Day,
  ctYDay                      :: Int,
  ctTZName                    :: String,
  ctTZ                        :: Int,
  ctIsDST                     :: Bool
}
```
### Parameterised types
```Haskell
data Maybe a = Just a
             | Nothing
```
### Recursive types
```Haskell
data List a = Cons a (List a)
            | Nil
              deriving (Show)
```
## streamlining function
### local variables
#### let
```Haskell
lend amount balance = let reserve    = 100
                          newBalance = balance - amount
                      in if balance < reserve
                         then Nothing
                         else Just newBalance
```
#### where
```Haskell
lend2 amount balance = if amount < reserve * 0.5
                       then Just newBalance
                       else Nothing
    where reserve    = 100
          newBalance = balance - amount
```
### case expression
```Haskell
fromMaybe defval wrapped =
    case wrapped of
      Nothing     -> defval
      Just value  -> value
```
### Conditional evaluation with guards
```Haskell
lend3 amount balance
     | amount <= 0            = Nothing
     | amount > reserve * 0.5 = Nothing
     | otherwise              = Just newBalance
    where reserve    = 100
          newBalance = balance - amount
```

# Type Class

## Monad
### Type constructor `m`
### Chain function `m a -> (a -> m b) -> m b`
### Inject function `a -> m a`

## Functor

## Monoid

## Traversable