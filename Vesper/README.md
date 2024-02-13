![uC_5K4](https://github.com/Samurai-Ebben/Portflio/assets/71189461/c558950a-43bb-4c30-9fd4-10eb1c9613a2)

## Summary of the project
Vesper was a 7 weeks game project that was pitched by my classmate at Yrgo 
## About

Vesper is a skill-based puzzle platformer where you play as a formless celestial being
that has been trapped in a physical form. Use the ability to change sizes to find your way home.

## Main Mechanics 
As a 2D fast-paced platformer, one of the most important main mechanics is the
*player movement*, *changing* sizes, and the *particle system*. I was mostly responsible for the implementation of these mechanisms. 

###   -  Player Movement
Of course, the player movement consumed most of my time to perfect it in a way that was as smooth as it is now;
this is how it began.
![before](/Vesper/Images/1stWeekMovement.gif)

However, after hard work and better code implementation;

![now](/Vesper/Images/nowMovement.gif)

The Player mechanics are the largest mechanic in the game. I covered the movement part, focusing on the input precision and the difference in 
stats of each size the player character has (Three: <Big, Medium, Small>).

The game was made with the **(new)Input system** in Unity. Using the Unity Events and *Delegates* to add functionality to the player. 
<details >
          <summary>Show Input functions</summary>

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
</details>


###   -  Swithching Sizes

<details>

          
 ```cs
  using System;
using System.Collections;
using System.Collections.Generic;
//using System.Drawing;
using UnityEngine;
using UnityEngine.InputSystem;

public enum Sizes { SMALL, MEDIUM, BIG };

public class PlayerController : MonoBehaviour
{
    public float vibrationDuration = .5f;
 
    public static PlayerController instance;

    // Size
    private     bool    isBig               =   false;
    private     bool    isSmall             =   false;

    // Player Controls
    [SerializeField] float      deacceleration  =   4;
    [SerializeField] float      acceleration    =   20;
    [SerializeField] float      maxSpeed        =   4;
    [SerializeField] float      velocityX;
    
    // Jump
    [Header("Jump Controls")]
    float       jumpBufferTime         =       0.1f;
    float       jumpHoldForce          =       5f;
    float       coyoteTime             =       0.05f;
    float       jumpCutOff             =       0.1f;
    float       jumpForce              =       6.0f;

    public float platformAvoidOnJumpOffset       =       10;
    public bool isJumping                        =       false;
    public bool jumpPressed                      =       false;
    bool canJump                                 =       true;

    float       coyoteTimer;
    float       jumpBufferTimer;
    public      bool    isBouncing;
    public      bool    canMove         =       true;

    public      bool    startedJump     =       false;
    public      bool    hasLanded       =       false;

    [Header("AirControls")]
    public bool inAir                   =       false;
    float       airSpeedMultiplier      =        .9f;
    float       airAccMultiplier        =        .9f;
    float       airDecMultiplier        =        .9f;

    // Ground Check
    [Header("Ground Check")]
    public float shortenGroundCheckXValue = 0.1f;

    [SerializeField] private    Vector2     groundCheckSize;
    [SerializeField] private    Transform   groundCheck;
    [SerializeField] private    LayerMask   layerIsGround;

    // Players properties
    public      Sizes               currentSize     { get; set; }
    public      Vector2             moveInput       { get; private set; }
    public      Rigidbody2D         rb              { get; private set; }
    public      InputActionAsset    actions         { get; private set; }
    public      AudioManager        audioManager    { get; private set; }
    public      RayCastHandler      rayCastHandler  { get; private set; }
    public      ScreenShakeHandler  screenShake     { get; private set; }
    public      PlayerParticleEffect effects        { get; private set; }
    
    // Player references
    DevButtons          devButtons;
    SizeStats           sizeStats;
    SquishAndSquash     squishAndSquash;

    public bool pausedPressed = false;

    private void Awake()
    {
        if (instance != null) return;
        instance = this;
        player = gameObject;

        actions = GetComponent<PlayerInput>().actions;
        sizeStats = GetComponent<SizeStats>();

        actions["Move"].performed += Move;
        actions["Move"].canceled += Move;

        actions["Pause"].performed += OnPause;
        actions["Navigate"].performed += OnNavigate;
        actions["Navigate"].started += OnNavigate;

        actions["Jump"].performed += OnJumpStarted;
        actions["Jump"].canceled += OnJumpCanceled;

        actions["Smaller"].started += Smaller;
        actions["Smaller"].canceled += SmallerCancel;

        actions["Larger"].started += Larger;
        actions["Larger"].canceled += LargerCancel;

        actions.Enable();
    }

    void Start()
    {
        devButtons          =     GameManager.Instance.gameObject.GetComponent<DevButtons>();
        squishAndSquash     =     GetComponentInChildren<SquishAndSquash>();
        effects             =     GetComponent<PlayerParticleEffect>();
        rb                  =     GetComponent<Rigidbody2D>();

        currentSize = Sizes.MEDIUM;
        jumpBufferTimer = 0;

        wallCollisionSquash = true;
        RegisterSelfToResettableManager();
    }

    void FixedUpdate()
    {
        MoveX();
        Jump();
        CoyoteEffect();
    }

    void Update()
    {
        JuiceFx();
        SwitchSize();
    }
    private void SwitchSize()
    {
        if (GameManager.Instance.GetComponent<PauseManager>().isPaused) return;

        if (isSmall && smallEnabled)
            currentSize = Sizes.SMALL;

        if (isBig && bigEnabled && rayCastHandler.largeTopIsFree && (rayCastHandler.anySide) && rayCastHandler.diagonalTop)
            currentSize = Sizes.BIG;

        if ((!isBig && !isSmall) && rayCastHandler.smallTopIsFree && (rayCastHandler.anySide) && rayCastHandler.diagonalTop) 
            currentSize = Sizes.MEDIUM;

        List<float> statList = sizeStats.ReturnStats(currentSize);

        transform.localScale    = new Vector3(statList[0], statList[0], statList[0]);
        maxSpeed                =       statList[1];
        acceleration            =       statList[2];
        deacceleration          =       statList[3];
        jumpForce               =       statList[4];
        rb.gravityScale         =       statList[5];
        jumpCutOff              =       statList[6];
        airSpeedMultiplier      =       statList[9];
        airAccMultiplier        =       statList[10];
        airDecMultiplier        =       statList[11];
        landingSFX              =       statList[12];
    }

    private void MoveX()
    {
        if (isBouncing || !canMove) return;

        if (inAir)
        {
            maxSpeed *= airSpeedMultiplier;
            acceleration *= airAccMultiplier;
        }
        else
        {
            maxSpeed /= airSpeedMultiplier;
            acceleration /= airAccMultiplier;
        }

        velocityX += moveInput.x * acceleration * Time.deltaTime;

        if (devButtons != null)
        {
            if (devButtons.amGhost)
            {
                float velocityY = 0;
                velocityY += moveInput.y * acceleration;
                rb.velocity = new Vector2(velocityX, velocityY);
            }
        }

        velocityX = Mathf.Clamp(velocityX, -maxSpeed, maxSpeed);

        if (moveInput.x == 0 || (moveInput.x < 0 == velocityX > 0))
        {
            if (inAir)
            {
                deacceleration *= airDecMultiplier;
            }
            velocityX *= 1 - deacceleration * Time.deltaTime;

            if (rb.velocity.magnitude < 0.1f)
            {
                velocityX = 0;
            }
        }
            
        rb.velocity = new Vector2(velocityX, rb.velocity.y);
    }

    void Jump()
    {
        if (!canJump || !jumpPressed || !canMove ) return;

        effects.CreateJumpDust();
        effects.StopLandDust();

        if (coyoteTimer > 0 && jumpBufferTimer > 0)
        {
            rb.velocity = Vector2.up * jumpForce;
            jumpBufferTimer = 0;
            isJumping = true;
            inAir = true;
            squishAndSquash.JumpSquash();

            if (!rayCastHandler.rightHelpCheck  && rayCastHandler.leftHelpCheck && rayCastHandler.centerIsFree)
                rb.velocity = new Vector2(rb.velocity.x + platformAvoidOnJumpOffset, rb.velocity.y);

            if (rayCastHandler.rightHelpCheck && !rayCastHandler.leftHelpCheck && rayCastHandler.centerIsFree)
                rb.velocity = new Vector2(rb.velocity.x - platformAvoidOnJumpOffset, rb.velocity.y);
        }

        else if (!isJumping && rb.velocity.y > 0)
            rb.velocity = new Vector2(rb.velocity.x, jumpHoldForce);

        jumpPressed = false;
    }

    void CoyoteEffect()
    {
        if (IsGrounded())
        {
            coyoteTimer = coyoteTime;
            canJump = true;
            
            isJumping = false;
        }
        else
        {
            coyoteTimer -= Time.deltaTime;
            if(coyoteTimer <= 0)
                canJump = false;
        }
    }

    #region Checkers
    public bool IsGrounded()
    {
        BoxCollider2D collider = GetComponentInChildren<BoxCollider2D>();
        Vector2 colliderSize = collider.size * new Vector2(1,0);
        Vector2 scaledColliderSize = new Vector2(colliderSize.x * transform.localScale.x, colliderSize.y * transform.localScale.y);
        Vector2 offset = new Vector2(shortenGroundCheckXValue, 0);
        groundCheckSize = scaledColliderSize - offset;

        return Physics2D.OverlapBox(groundCheck.position, groundCheckSize, 0, layerIsGround);
    }
    #endregion
    
    #region InputHanlder

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

    public void Smaller(InputAction.CallbackContext ctx)
    {
        isSmall = true;
    }
    public void SmallerCancel(InputAction.CallbackContext ctx)
    {
        isSmall = false;
    }
    public void Larger(InputAction.CallbackContext ctx)
    {
        isBig = true;
    }
    public void LargerCancel(InputAction.CallbackContext ctx)
    {
        isBig = false;
    }

    public void OnPause(InputAction.CallbackContext ctx)
    {
        GameManager.Instance.GetComponent<PauseManager>().BackMenu();
        GameManager.Instance.GetComponent<PauseManager>().PauseTrigger();
    }

    public void OnControls(InputAction.CallbackContext ctx)
    {
        GameManager.Instance.GetComponent<PauseManager>().ControlsMenu();
    }

    public void OnNavigate(InputAction.CallbackContext ctx)
    {
        //checks if vertical.
        if(MathF.Abs(ctx.ReadValue<Vector2>().y) > 0 && GameManager.Instance.GetComponent<PauseManager>().isPaused)
        {
            AudioManager.Instance.GameplaySFX(AudioManager.Instance.clickInMenu, AudioManager.Instance.clickInMenuVolume);
        }
    }
    private void OnDisable()
    {
        actions["Move"].performed -= Move;
        actions["Move"].canceled -= Move;

        actions["Jump"].performed -= OnJumpStarted;
        actions["Jump"].canceled -= OnJumpCanceled;
        actions["Navigate"].performed -= Move;

        #region switchControls
        actions["Smaller"].started -= Smaller;
        actions["Smaller"].canceled -= SmallerCancel;
        actions["Larger"].started -= Larger;
        actions["Larger"].canceled -= LargerCancel;
        #endregion

        actions.Disable();
    }
    #endregion

    public void VibrateController(float lowFreq, float highFreq, float duration)
    {
        if (gPad != null)
        {
            print(gPad.name);

            gPad.SetMotorSpeeds(lowFreq, highFreq);
            StartCoroutine(StopViberation(duration, gPad));

        }
    }

    IEnumerator StopViberation(float duration, Gamepad pad)
    {
        float elabsedTime = 0;
        while(elabsedTime < duration)
        {
            elabsedTime += Time.deltaTime;
            yield return null;
        }

        pad.SetMotorSpeeds(0,0);
    }

    private void OnDrawGizmosSelected()
    {
        Gizmos.color = Color.yellow;
        Gizmos.DrawWireCube(groundCheck.position, groundCheckSize);
        Gizmos.color = Color.red;
    }

    private void OnDestroy()
    {
        actions["Pause"].performed -= OnPause;
    }

    public void Reset()
    {
        currentSize = Sizes.MEDIUM;
    }
}
  ```
</details>
