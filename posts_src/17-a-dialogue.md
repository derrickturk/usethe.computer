% a dialogue

<img class="centered" alt="Socrates and Plato" src="/images/plato_socrates.jpg" height="600px">

SOCRATES: what is "machine learning"?  
PLATO: why, the improvement of an algorithm's abilities by the examination of data.  
SOCRATES: so, then, a linear regression on a computer is machine learning?  
PLATO: quite so. a candle is not the Sun, but does it not yet illuminate?  
SOCRATES: how shall I judge whether the machine has learned?  
PLATO: we shall draw some plots.  
SOCRATES: what, then, is a plot?  
PLATO: why, a change of coordinates.  

```python
from sys import stdout

def plot(series, width=80, height=24):
    def change_coords(s):
        x_min = min(p[0] for p in s)
        x_max = max(p[0] for p in s)
        y_min = min(p[1] for p in s)
        y_max = max(p[1] for p in s)
        return [(
          int((p[0] - x_min) / (x_max - x_min) * (width - 1)),
          int((p[1] - y_min) / (y_max - y_min) * (height - 1))
        ) for p in s]
    to_plot = dict()
    for char, s in series.items():
        for coord in change_coords(s):
            if coord in to_plot:
                to_plot[coord] += char
            else:
                to_plot[coord] = char
    for i in range(height):
        for j in range(width):
            chars = to_plot.get((j, height - 1 - i))
            if chars is not None:
                if len(chars) > 1:
                    stdout.write('?')
                else:
                    stdout.write(chars)
            else:
                stdout.write(' ')
        stdout.write('\n')

xs = list(range(10))
ys = [x ** 2 for x in xs]
plot({'o': list(zip(xs, ys))})
```

<!-- TODO: blockquote -->
```
                                                                               o
                                                                                
                                                                                
                                                                                
                                                                                
                                                                      o         
                                                                                
                                                                                
                                                                                
                                                                                
                                                             o                  
                                                                                
                                                                                
                                                    o                           
                                                                                
                                                                                
                                           o                                    
                                                                                
                                                                                
                                   o                                            
                                                                                
                          o                                                     
                 o                                                              
o       o                                                                       
```

SOCRATES: quite so. how, then, shall the machine "learn"?  
PLATO: by minimizing error.  
SOCRATES: and what is error?  
PLATO: why, the difference between what is expected and what is found.  
SOCRATES: and how shall we minimize the difference between what is expected and what is found?  
PLATO: by rolling down the hill.  
SOCRATES: and how do we know which way is "down"?  
PLATO: by considering the rate of change.  
SOCRATES: what is the rate of change?  
PLATO: the rate of change is the ratio of how much this portion of hill changes in elevation to the size of this portion of the hill.  

```python
def minimize(f, guess, eps=1e-8, rate=1e-3, max_iter=10000):
    def approx_f_prime(guess, f_guess):
        f_prime = (f(guess + eps) - f_guess) / eps
        return f_prime
    for _ in range(max_iter):
        f_guess = f(guess)
        f_prime = approx_f_prime(guess, f_guess)
        new_guess = guess - rate * f_prime
        if abs(guess - new_guess) < eps:
            return new_guess
        guess = new_guess
    return None

f = lambda x: (x - 2) ** 2
print(minimize(f, 0.0))
```

> `1.9999950134026048`

SOCRATES: what shall we learn?  
PLATO: let us find an approximation of a data set by a exponential function y = e^kx.  
SOCRATES: which is to say, let us find the value of k which minimizes the error?  
PLATO: quite.  
SOCRATES: so, first, we must define the difference between what is expected and what is found?  
PLATO: exactly so.  

```python
from math import exp

def model(k, x):
    return exp(k * x)

def sse(k, data):
    return sum((model(k, x) - y) ** 2 for x, y in data)

NOISY_DATA = [
    (1.0, 2.1),
    (2.0, 4.6),
    (3.0, 9.3),
    (4.0, 20.7),
    (5.0, 41.4),
]

plot({'o': NOISY_DATA})
print(sse(1.0, NOISY_DATA))
print(sse(0.5, NOISY_DATA))
```

```
                                                                               o
                                                                                
                                                                                
                                                                                
                                                                                
                                                                                
                                                                                
                                                                                
                                                                                
                                                                                
                                                                                
                                                                                
                                                                                
                                                           o                    
                                                                                
                                                                                
                                                                                
                                                                                
                                                                                
                                       o                                        
                                                                                
                                                                                
                   o                                                            
o                                                                               
```
> `12725.389709846757`  
> `1057.8045226772697`  

SOCRATES: I see that you are trying to pull a fast one on me, PLATO. you have taken the square of the difference between that which is expected and that which is found.  
PLATO: there is no deception intended - it is to make the hill smoother.  
SOCRATES: so that we may roll down it more easily?  
PLATO: quite so.  

```python
best_k = minimize(lambda k: sse(k, NOISY_DATA), 0.5, rate=1e-5)
print(best_k)
```

SOCRATES: again I suspect deception - for you have changed the speed with which we roll down the hill.  
PLATO: some hills are steeper than others.  
SOCRATES: I remain skeptical. nonetheless, how shall we judge whether we have learned well?  
PLATO: we shall compare that which is expected with that which is found.  

```python
print(sse(best_k, NOISY_DATA))
plot({'o': NOISY_DATA, 'x': [(x, model(best_k, x)) for x, _ in NOISY_DATA]})
```

> `0.9738838331094835`  

```
                                                                               ?
                                                                                
                                                                                
                                                                                
                                                                                
                                                                                
                                                                                
                                                                                
                                                                                
                                                                                
                                                                                
                                                                                
                                                                                
                                                           ?                    
                                                                                
                                                                                
                                                                                
                                                                                
                                                                                
                                       ?                                        
                                                                                
                                                                                
                   ?                                                            
?                                                                               
```

SOCRATES: from the question marks I infer that which is found is about the same as that which is expected, given the change of coordinates.  
PLATO: exactly.  
SOCRATES: you have said that for an algorithm to learn is to improve it's abilities by the examination of data.  
PLATO: that is so.  
SOCRATES: you have shown that your algorithm's abilities have improved at the task of repeating the data it has examined. but what of its abilities to make predictions at new, unexamined values? we do not consider the mockingbird the greatest singer among birds.  
PLATO: you speak of generalization.  
SOCRATES: I speak of utility, for of what value is an algorithm which can be replaced by a mockingbird?  
PLATO: that which provides this utility, I call generalization.  
SOCRATES: and how may we judge whether an algorithm is general?  
PLATO: by testing against that which is known to us, but not examined by the algorithm, and again comparing what is expected to what is found.  
```python
def leave_one_out(data):
    for i in range(len(data)):
        train = [d for j, d in enumerate(data) if i != j]
        test = [data[i]]

        best_k = minimize(lambda k: sse(k, train), 0.5, rate=1e-5)
        print(f'k = {best_k}')
        print(f'test error = {sse(best_k, test)}')

leave_one_out(NOISY_DATA)
```

> `k = 0.7462941022745024`  
> `test error = 8.407331581956807e-05`  
> `k = 0.7462671426037844`  
> `test error = 0.022996313561218864`  
> `k = 0.7463403623416127`  
> `test error = 0.0070796292554830996`  
> `k = 0.7446659508933553`  
> `test error = 1.078425294824736`  
> `k = 0.756065110727205`  
> `test error = 5.90639954833926`  

SOCRATES: I see that in each case, we arrive at almost the same model.  
PLATO: and thus we believe that the algorithm generalizes.  
SOCRATES: this has been most enlightening. now, may we say that this is "artificial intelligence"?  
PLATO: well...  
DIOGENES: get the hell away from my trash can! what's a guy got to do to get a little peace and quiet with all these roving philosphers around?

---

All code is available <a href="https://gist.github.com/derrickturk/a22e2300821c8740a34f49c553928c94">here</a>.
