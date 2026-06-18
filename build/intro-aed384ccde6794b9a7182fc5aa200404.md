# Financial Python
## Volume: Pricing And Interest Rate Risk
### Chapter Two: Par Yields And The Term Structure Of Interest Rates


A par yield defines the theoretical coupon rate required for a newly issued, default-free bond to price exactly at par.  At first thought you might think that par yield is a characteristic of an individual bond, but it's not. In reality,  par yields are just the term structure in disguise—a mathematical reshuffling of discount factors used to piece together purely hypothetical, imaginary bonds,


#### The Mathematics of the "Imaginary Bond"

If we assume a par value of $1$ (or $100\%$), the price of our imaginary par bond is exactly $1$. The cash flows consist of regular coupon payments ($c$) multiplied by the accrual fraction ($\gamma_i$), plus the return of the principal at maturity ($T$).
When you set the present value of those cash flows equal to $1$ using the economy's discount factors ($PV$), you get:
<br>

 $$1 = c \sum_{i=1}^{T} \gamma_i PV(t) + PV(T)$$
<br>

What happens when you solve for c? a pure rearrangement of the present value factors:
<br>

 $$c = \frac{1 - PV(T)}{\sum_{i=1}^{T} \gamma_i PV(t)}$$
 
#### Why Build This Construct?

Even though no real-world bond perfectly matches this curve , constructing this imaginary curve serves several purposes:

*  **Standardization**: It provides an apples-to-apples baseline.
   *  Traders can look at the par yield curve to instantly know the "fair" coupon to attach to a brand-new primary market issuance today. 
   *  Corporations can use it to benchmark the cost of their existing or future debt they intend to issue.
   *  Investors can use it to benchmark current return opportunities.
*  **Intuitive Clarity**: It is far more natural to conceptualize fixed-income investments through standard maturities and coupon rates rather than abstract present value factors and spot rates, giving par yields a distinct and easily relatable appeal
*  **Bootstrapping the Curve**: As demonstrated in this Chapter, published par yields can be used to bootstrap the term structure.
