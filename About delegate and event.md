# About C# Delegate And Event 
## Delegate
1. internal delegate is a class (inherite form System.MulticastDelegate)

2. delegate support both covariance and contravariance internal

3. when using delegate chian you create a new instance

4. when using lambda which reference local value will create new class

5. when using lambda will add new static instance and method internal.
## Event
1. event is based on delegate

2. event is deisgend for *"event pattern"*

3. when using event you should define a class "xxxEventArgs" base on "eventArgs"

4. add event handler 

5. when triger event should using Volatile.Read to check null under concurrent cases.

6. compiler will add two public methods internal ("add_xxx" for += and "remove_xxx" for -= ) which will use do while and interlocked.compareExchange to build/remove delegate chain in thread safe.

7. when you do not want to receive you should use -= to remove it.