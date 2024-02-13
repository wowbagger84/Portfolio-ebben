![uC_5K4](https://github.com/Samurai-Ebben/Portflio/assets/71189461/c558950a-43bb-4c30-9fd4-10eb1c9613a2)

## Summary of the project
Vesper was a 7 weeks game project that was pitched by my classmate at Yrgo 
## About

Vesper is a skill-based puzzle platformer where you play as a formless celestial being
that has been trapped in a physical form. Use the ability to change sizes to find your way home.

## Main Mechanics 
As a 2D fast-paced platformer, one of the most important main mechanics is the
*player movement*, *changing* sizes, and the *particle system*. I was mostly responsible for the implementation of these mechanisms. 

###   - Player Movement
Of course, the player movement consumed most of my time to perfect it in a way that was as smooth as it is now;
this is how it began.
![before](/Vesper/Images/1stWeekMovement.gif)

However, after hard work and better code implementation;

![now](/Vesper/Images/nowMovement.gif)

The Player mechanics are the largest mechanic in the game. I covered the movement part, focusing on the input precision and the difference in 
stats of each size the player character has (Three: <Big, Medium, Small>).

The game was made with the **(new)Input system** in Unity. Using the Unity Events and *Delegates* to add functionality to the player. 
```CS
public void Move(InputAction.CallbackContext ctx)
{
        moveInput = ctx.ReadValue<Vector2>();
}

void OnJumpStarted(InputAction.CallbackContext ctx)
{
     startedJump = true;

     if (ctx.performed)
      {
            jumpBufferTimer = jumpBufferTime;
            jumpPressed = true;
        }
    }

void OnJumpCanceled(InputAction.CallbackContext ctx)
{
        jumpBufferTimer -= jumpBufferTime;
        if (!ctx.performed && rb.velocity.y > 0)
        {
            rb.velocity = new Vector2(rb.velocity.x, rb.velocity.y * jumpCutOff);
            coyoteTimer = 0;
        }
}
```
Calling these functions using delegate looked like this:
```cs
actions["Jump"].performed += OnJumpStarted;
actions["Jump"].canceled += OnJumpCanceled;
```
