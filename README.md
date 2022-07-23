# Player-controller
// This is a simple player controller for my future project.
// Controls are : (WASD) or (Right Mouse) to move the character in the world, (Left Mouse) to shoot, (Shift) to run, double click (C) to dash in a direction, hold (F) while moving to move slower.
// I've made the controller pretty solid so there are almost no bugs at this point. The player can slide in 4 directions (north, south, west and east), when a slope is >= 45 degrees. Most of my time with this small project was making it a solid controller with clean and readable code.
// if the project can't be opened here is a text based version of the player controls, all of it isn't in this piece:


using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class Movement : MonoBehaviour // takes care of all the movement specific things of the player (+ Basic shoot)
{
    [SerializeField] private Animator _animator; // for future basically, if anything needs animating this does the job. althought we might want the animations in a different script to keep things organized and clean
    private CharacterController _controller; // letting this script use the CharacterController class
    private PlayerShoot shootScript; // used in this script to use the shoot method
   
    // bools

    private bool isSprinting = false; // used to disable movement speed stacking
    private bool isSneaking = false; // used to disable movement speed stacking
    private bool _isDashing; // used disable movements and shooting atm if the player is dashing

    private bool defaultControl = true; // defaults the dash direction to facing direction, if the player isnt pressing wasd keys
    private bool mouseIsUsed; // checks to see if mouse(1) is being used to stop player speed from multiplying
    private bool wasdIsUsed; // checks to see if w,a,s or d are being pressed to stop player speed from multiplying

    private bool wasdControlTrue = true; // its a check to see which controll the player uses, so that the speed from mouse movement and wasd movement dont stack (X2)
    // floats and ints (values)   
    public float speedModificator = 0; // holds a value that adjusts the players moving speed. can be modified in other scripts to give the player for example:(speed boost or slowness debuff)
    [SerializeField] private float _speed = 1f;  // basic moving/running speed
    [SerializeField] private float _sprintSpeed = 3.5f; // sprint speed adjuster. i dont feel like this should be a thing, atleast not for all classes. maybe this could be an ability for a rogue?
    [SerializeField] private float _sneakSpeed = -2f; // sneak / crouch / walking speed adjuster

    private float dashDirection; // the value of a direction the player is dashing to
    [SerializeField] private float dashCooldown = 5f; // a cooldown timer for the dash ability
    private float doublePressCooldown = 0.0f; // // removing these probably. these were used for the dash to check if the played double tapped a button to dash
    private float doublePressCount; // removing these probably. these were used for the dash to check if the played double tapped a button to dash

    [SerializeField] private float ySpeed = -0.13f; // basically value of gravity
    private float _groundRayDistance = 1; // raycast distance
    private float slopeSlideSpeed = -4f; // sliding speed value
    private int slopeLockValue = 0; // holds the value of the direction which the player is supposed to face, when sliding

    // Vector3's
    private Vector3 movement; // holds the value of the wasd movement directions
    public Vector3 mouseMovement; // holds the value of the mouse movement direction
    private Vector2 mouseOnScreen; // holds the value for the mouse position on the screen
    Vector2 positionOnScreen; // holds the value for the player position on the screen
    [SerializeField] private Vector3 lastPosition; // when player starts sliding it records the lastposition before slide start and locks the player to face sliding direction

    // Raycasts
    private RaycastHit _slopeHit; // checks the ground slopes
    
    // Methods
    private void Start() 
    {
        shootScript = GetComponentInChildren<PlayerShoot>();
        _animator = GetComponentInChildren<Animator>();
        _controller = GetComponent<CharacterController>();

       
    }
    
    private void LateUpdate()
    {
        if (slopeLockValue != 1)
        {

            lastPosition = transform.position;
        } // this needs to update after update so that the slope lock mechanism works
    }

    private void Update() 
    {
        Timers();
        
        // Debug.Log(dashDirection);
        // Debug.DrawRay(transform.position + new Vector3(0, -0.08f, 0.15f), Vector3.down, Color.green, (_controller.height / 2f) + _groundRayDistance);
        //Debug.DrawRay(transform.position + new Vector3(0, -0.08f, -0.15f), Vector3.down, Color.green, (_controller.height / 2f) + _groundRayDistance);
        //Debug.DrawRay(transform.position + new Vector3(0.15f, -0.08f, 0), Vector3.down, Color.green, (_controller.height / 2f) + _groundRayDistance);
        //Debug.DrawRay(transform.position + new Vector3(-0.15f, -0.08f, 0), Vector3.down, Color.green, (_controller.height / 2f) + _groundRayDistance);

        // make the player rotate around with the mouse. // also need to add position. follow mouse position


        
        if (!OnSteepSlope())
        {
            if (!_isDashing)
            {
                MovingModificators();
                PlayerRotation(); // when on slope, we need the player  lock to face the sliding direction
                IsDashing();
                slopeLockValue = 0;
                if (Input.GetMouseButtonDown(0))
                {
                  
                    // if value ++ && value < 3 { check if player shoots again with a timer, if not, then reset value & timer }
                    // value ++; until 3. if value == 3 { cooldownTimer };
                    // if reloadtime == 0;
                    if(shootScript.cooldownTimer <= 0)
                    {
                        shootScript.ShootBullet();
                    }
                    
                }

            }

        } // enables features such as : moving, player rotation, certain animations, (etc.), when not on a highly angled slope.
       
   
        
       
        
            MovePlayer();
        
        


        //  Debug.Log(Vector3.Angle(_slopeHit.normal, Vector3.up));
        //Debug.Log(Vector3.up - _slopeHit.normal * Vector3.Dot(Vector3.up, _slopeHit.normal)+ "asd");
    } 

    private void MovePlayer()
    {
        // !! KEEP HIERARCHY AS IS. Code depends on a hierarchy to work properly, so moving stuff might break adjustments.
        

     

        float x = Input.GetAxisRaw("Horizontal"); // -A and +D vector.left-/right+
        float z = Input.GetAxisRaw("Vertical"); // +W and -S vector.back-/forward+

        mouseMovement = transform.TransformDirection(Vector3.left); // movementDirection
        movement = new Vector3(x, 0, z).normalized; // movementDirection

        Vector3 verticalMovement = Vector3.up * ySpeed;

        if (OnSteepSlope())
        {
            StopCoroutine(Dash());
            _isDashing = false;
            speedModificator = 0; // modificator stands for running and crouching , but also for stat debuffs and buffs. probably dont need this here. investigate.
            SteepSlopeMovement();
            slopeLockValue = 1;
            SlideRotation();
            _controller.Move(verticalMovement + movement * (_speed + speedModificator) * Time.deltaTime);
           
        }

        mouseIsUsed = Input.GetMouseButton(1);
        wasdIsUsed = Input.GetKey(KeyCode.W)||Input.GetKey(KeyCode.S) || Input.GetKey(KeyCode.A) || Input.GetKey(KeyCode.D);

        if (wasdIsUsed)
        {
            wasdControlTrue = true;
        }

        if(mouseIsUsed && !wasdIsUsed)
        {
            wasdControlTrue = false;
        }

        if (wasdControlTrue )
        {
            
            _controller.Move(verticalMovement + movement.normalized * (_speed + speedModificator) * Time.deltaTime);
        } // active wasd

        if (!wasdControlTrue )
        {
          
          
           
            _controller.Move(verticalMovement);
            if (Input.GetMouseButton(1))
            {
                
                if (!OnSteepSlope())
                {
                    if (!_isDashing)
                    {
                        _controller.Move(mouseMovement * (_speed + speedModificator) * Time.deltaTime); // im happy, but pissed at the same time. i played around with rotations,angles,convertions. it took me a whole day to do this shit:))))
                    }
                   

                }
                
            }
        } // active mouse
    } // takes care of wasd and mouse movements

    private void MovingModificators()
    {
        isSneaking = Input.GetKey(KeyCode.F); // control brings up a thing if while playing you press ctrl left and s
        isSprinting = Input.GetKey(KeyCode.LeftShift);

        if (isSneaking && !isSprinting)
        {
            speedModificator = _sneakSpeed;
        }
        if (isSprinting && !isSneaking)
        {
            speedModificator = _sprintSpeed;
        }
        if (!isSneaking && !isSprinting || isSneaking && isSprinting)
        {
            speedModificator = 0;
        }
        // example : enemy.Slow(-10)-> modificator -= 10 -> slow decayed modificator += 10
        // adds or subtracts from speed according to the modificator
    } // takes care of players speed modifications : (sprint, walk, slowness debuff, speed buff)

    private void PlayerRotation() 
    {
         positionOnScreen = Camera.main.WorldToViewportPoint(transform.position);

        mouseOnScreen = (Vector2)Camera.main.ScreenToViewportPoint(Input.mousePosition);

        float angle = AngleBetweenTwoPoints(positionOnScreen, mouseOnScreen);
       
        
        transform.rotation = Quaternion.Euler(new Vector3(0f, -angle, 0f));

        float AngleBetweenTwoPoints(Vector3 a, Vector3 b)
        {
            return Mathf.Atan2(a.y - b.y, a.x - b.x) * Mathf.Rad2Deg;
            
        }
            
    } // make the player face the mouse direction   

    private bool OnSteepSlope()
    {
        if (!_controller.isGrounded) return false;

        if (Physics.Raycast(transform.position + new Vector3(0, -0.08f, 0.15f),Vector3.down, out _slopeHit, (_controller.height / 2f) + _groundRayDistance)|| 
            Physics.Raycast(transform.position + new Vector3(0, -0.08f, -0.15f), Vector3.down, out _slopeHit, (_controller.height / 2f) + _groundRayDistance)||
            Physics.Raycast(transform.position + new Vector3(0.15f, -0.08f, 0), Vector3.down, out _slopeHit, (_controller.height / 2f) + _groundRayDistance)||
            Physics.Raycast(transform.position + new Vector3(-0.15f, -0.08f, 0), Vector3.down, out _slopeHit, (_controller.height / 2f) + _groundRayDistance))
        {
            float _slopeAngle = Vector3.Angle(_slopeHit.normal, Vector3.up);
            if (_slopeAngle > _controller.slopeLimit) return true;



        } // 4 raycasts in 4 directions of the player, checking the floor, if there is an angle of 45 degrees under the player. if there is, player slides to the direction the angle is 
        return false;
    } // uses raycasts and GameObject rotation angles to check, if the player is on a slope >= (45 degrees or more = steepslope).

    private void SteepSlopeMovement()
    {
        Vector3 slopeDirection = Vector3.up - _slopeHit.normal * Vector3.Dot(Vector3.up, _slopeHit.normal); // got this from a tutorial. raycast laser check the angle of the ground and the direction of the slope pretty much
        float slideSpeed = _speed + slopeSlideSpeed + Time.deltaTime;
        movement = slopeDirection * -slideSpeed;

        movement.y = movement.y - _slopeHit.point.y;
    } // locks the player moving direction to the "sliding" direction when on steep slopes

    private void SlideRotation()
    {
        if (slopeLockValue == 1) // using these values we can create a coroutine that pushes the player towards the correct slide direction, once slide ends
        {
            transform.rotation = Quaternion.Euler(0, 0, 0); // reset all turns
            if (transform.position.x > lastPosition.x && lastPosition.z == transform.position.z)
            {
                transform.rotation = Quaternion.Euler(0, 180, 0); // Right
                

            }
            if (transform.position.x < lastPosition.x && lastPosition.z == transform.position.z)
            {
                transform.rotation = Quaternion.Euler(0, 0, 0); // Left // we can turn the camera to phase the right correction which means forward would be 0,0,0 and back would be 0, 180, 0 and not the other way around. ill fix it later i got other stuff
                

            }
            if (transform.position.z > lastPosition.z && lastPosition.x == transform.position.x)
            {
                transform.rotation = Quaternion.Euler(0, 90, 0); // Forward
               

            }
            if (transform.position.z < lastPosition.z && lastPosition.x == transform.position.x)
            {
                transform.rotation = Quaternion.Euler(0, -90, 0); // Back
               

            }

        }
    } // locks the player facing direction to the sliding direction, when on steep slopes

    
   private void IsDashing()
    {

        // if shift is pressed while right click down and no wasd pressed: dash mouse dir
        // if shift is pressed while right click down and wasd is pressed: dash wasd dir

        bool w = Input.GetKey(KeyCode.W);
        bool s = Input.GetKey(KeyCode.S);
        bool d = Input.GetKey(KeyCode.D);
        bool a = Input.GetKey(KeyCode.A);
        if (w || s || d || a)
        {
            defaultControl = false;      
           
        }
        else if(!_isDashing)
        {
            defaultControl = true;
            dashDirection = 0;
        }

        // also can make a bool that disables during a cutscene or when inventory is open. add !Dashing() to movement enable



        if (Input.GetKeyDown(KeyCode.C) && dashCooldown <= 0) //  if double tapped
        {
         if(doublePressCooldown <= 0)
            {
                doublePressCooldown = 0.4f;
                doublePressCount = 0;
            }  
         if(doublePressCooldown > 0)
            {
                doublePressCount++;
                if(doublePressCount >= 2)
                {
                    if (w && !d && !a)
                    {

                        dashDirection = 1;

                    }// up
                    if (s && !d && !a)
                    {

                        dashDirection = 2;

                    }// down
                    if (d && !w && !s)
                    {

                        dashDirection = 3;

                    }// right
                    if (a && !w && !s)
                    {

                        dashDirection = 4;

                    }// left
                    if(w && d)
                    {
                        dashDirection = 5;


                    } // top right
                    if(w && a)
                    {

                        dashDirection = 6;

                    } // top left
                    if(s && d)
                    {

                        dashDirection = 7;

                    } // down right
                    if(s && a)
                    {
                        dashDirection = 8;


                    } // down left
                    Debug.Log("dash");
                   
                    dashCooldown = 5f;
                    doublePressCount = 0;
                    StartCoroutine(Dash());
                }
               
            }
         
            


         
        }
       

    } // does the check for player inputs to see if the player is supposed to dash
    
    private IEnumerator Dash()
    {
       

        for (float dashTime = 3f; dashTime >= 0; dashTime-= 0.1f)
        {
            
            
            if (!OnSteepSlope())
            {   
                if(dashTime > 0.1f)
                {
                    _isDashing = true;
                }
                else
                {
                   _isDashing = false;
                }

                // Vector3 verticalMovement = Vector3.up * ySpeed;
                //if mouse or wasd controlled dash
               if(defaultControl == false)
                { 
                    switch (dashDirection)
                    {
                        case 1: // Up
                            _controller.Move(Vector3.forward * (_speed * 3f) * Time.deltaTime);
                            break;
                        case 2: // Down
                            _controller.Move(Vector3.back * (_speed * 3f) * Time.deltaTime);
                            break;
                        case 3: // Right
                            _controller.Move(Vector3.right * (_speed * 3f) * Time.deltaTime);
                            break;
                        case 4:// Left
                            _controller.Move(Vector3.left * (_speed * 3f) * Time.deltaTime);
                            break;
                        case 5:// Top Right
                            _controller.Move((Vector3.forward / 2 + Vector3.right /2) * (_speed * 3f) * Time.deltaTime);
                            break;
                        case 6:// Top Left
                            _controller.Move((Vector3.forward / 2 + Vector3.left / 2) * (_speed * 3f) * Time.deltaTime);
                            break;
                        case 7:// Down Right
                            _controller.Move((Vector3.back / 2 + Vector3.right / 2) * (_speed * 3f) * Time.deltaTime);
                            break;
                        case 8:// Down Left 
                            _controller.Move((Vector3.back / 2 + Vector3.left / 2) * (_speed * 3f) * Time.deltaTime);
                            break;
                    }
                }
                if(defaultControl == true)
                {
                    _controller.Move(mouseMovement * (_speed * 3f)* Time.deltaTime);
                }
               

                yield return null;
            }
           
        }


    } // makes the dash happen to the correct direction, based on input

    private void Timers()
    {
        if (dashCooldown > 0)
        {
            dashCooldown -= Time.deltaTime;
        }
       if(doublePressCooldown > 0)
        {
            doublePressCooldown -= Time.deltaTime;
        }
       if(shootScript.shootTimer > 0)
        {
            shootScript.shootTimer -= Time.deltaTime;
        }
    }// takes care of all the timers in the script. if they need to be paused for some reason, its possible
}
