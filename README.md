# Unity Reference

## Introduction
For the past two weeks, I have been working on a sprint with the Tech Academy. Working with other dev's we put together a classic arcade game application in Unity. 
Each of us worked on a seperate classic arcade game, from 1942 to Pacman. I spent the two weeks putting together a missile command game.
I learned a lot during this sprint, from implmenting UI, to scene loading, to creating levels.
At the start of the week, all I had were a few empty scenes and an idea of what I wanted to do. By the end I had a fully functioning game.

Below I'll go over the stories I worked on and showcase some of the code and features I made during that time.

## Story 1: Basic Scenes
In this first story, I created three scenes for my game, and hooked them together with a scene loader.
This scene loader was provided to me for the projet, along with documention on how to use it. Thanks to the good documention, I was able to figure out how to use the scene loader and implement it in the game.
Thanks to the scene loader, this story didn't require any coding.

The main menu scene
![Main menu scene](assets/images/image1.png)

Gameplay scene
![Gameplay Scene](assets/images/image2.png)

Game over and credits scene
![Game over and credits scene](assets/images/image2.png)

## Story 2: Player behavior
With my scenes set up, it was now time to create the scripts for my player.

First, my player needed to be able to fire missiles when they left clicked on the screen.
I wanted to be able to use a pointed missile that could point towards the target, so some rotation math had to be put in.
I also added a stockpile of missiles that would be slowly regenerated as the player went on. A game manager kept track of that and wouldn't let the player fire missiles if they had run out.
```
public class MissileFire : MonoBehaviour
{
    public Transform firePoint;
    public GameObject missilePrefab;
    public GameObject GameManager;
    public AudioSource playerAudio;
    private Camera cam;

    private void Start()
    {
        cam = Camera.main;
    }

    void Update()
    {
        //Fires a missile when left click is made. Will only fire if there are missiles left
        if (Input.GetMouseButtonDown(0) && GameManager.GetComponent<MissileGameManager>().AreThereMissilesLeft() == true)
        {
            Vector2 direction = ClickPos() - (Vector2)transform.position;
            var angle = Mathf.Atan2(direction.y, direction.x) * Mathf.Rad2Deg;
            GameObject missile = Instantiate(missilePrefab, firePoint);
            missile.transform.rotation = Quaternion.Euler(new Vector3(0, 0, angle));
            playerAudio.PlayOneShot(playerAudio.clip);
            GameManager.GetComponent<MissileGameManager>().FiredMissile();
        }
    }

    //This method gets the point on the screen (reletive to the cammera) where the mouse is.
    //This should only be used with a left click, otherwise wierd behavor might happen.
    public Vector2 ClickPos()
    {
        Vector2 mousePos = Input.mousePosition;
        return cam.ScreenToWorldPoint(mousePos);
    }
}
```

After that, I had to create a player missile that would move around and delete itself.
In a future story, I also added instantion for a player explosion. However, for right now the player missile just deletes itself when it reaches where it was targeted.
```
public class MissileMove : MonoBehaviour
{
    private MissileFire missileFire;
    private Vector2 mousePos;
    private float speed = 10f;
    public GameObject explosion;
    // Start is called before the first frame update
    void Start()
    {
        //Gets the target location
        missileFire = FindObjectOfType<MissileFire>();
        mousePos = missileFire.ClickPos();
    }

    // Update is called once per frame
    void Update()
    {
        float step = speed * Time.deltaTime;

        //Moves the missile to the target location smoothly and keeps track of the current pos.
        transform.position = Vector2.MoveTowards(transform.position, mousePos, step);
        Vector2 currentPos = new Vector2(transform.position.x, transform.position.y);

        //Once target location is reached, spawns an explosion and destroys the missile.
        if (currentPos == mousePos)
        {
            Instantiate(explosion, currentPos, Quaternion.identity);
            Destroy(gameObject);
        }
    }
}
```

## Story 3: Opposition
This was likely one of the most challenging stoies I worked on during this sprint.
During this story, I had to code in my enemy missiles so that they could attack the various cities I had for the player to defend.
I ended up going for an array/list system. An enemy missile manager would have an array of empty game object spawners off screen and a list of the cities.
When it came time to spawn an enemy missile, a random spawner and city would be picked, then the enemy missile would be spawned.
In the next story I also added a round counter that would increase the number of spawns that would happen at the same time to make the game harder over time. In this story only one missile would spawn at a time.
```
public class EnemyMissileManager : MonoBehaviour
{
    public GameObject[] missileSpawners;
    public GameObject cityParent;
    public GameObject enemyMissile;
    public GameObject GameManager;
    public float spawnTimer = 4.0f;
    public List<Vector2> targetCities = new List<Vector2>();

    // Update is called once per frame
    void Update()
    {
        //Timer for the spawn
        if (spawnTimer > 0)
        {
            spawnTimer -= Time.deltaTime;
        }
        else
        {
            //Gets a random spawner, spawns an enemy missile there, then gets a random city to target and passes that to the missile before reseting the timer
            GetCities();
            SpawnMissile();
            spawnTimer = 3.0f;
        }
    }

    //If there are cities left, spawns a number of missiles equal to the current round. Each missile is spawned, passed a target city, and rotated to face it
    private void SpawnMissile()
    {
        if (targetCities.Count != 0)
        {
            for (int i = 0; i < GameManager.GetComponent<MissileGameManager>().GetRound(); i++)
            {
                var position = Random.Range(0, (missileSpawners.Length - 1));
                GameObject missile = Instantiate(enemyMissile, missileSpawners[position].transform.position, Quaternion.identity);
                missile.GetComponent<EnemyMissile>().target = TargetCity();
                Vector2 direction = missile.GetComponent<EnemyMissile>().target - transform.position;
                var angle = Mathf.Atan2(direction.y, direction.x) * Mathf.Rad2Deg;
                missile.transform.rotation = Quaternion.Euler(new Vector3(0, 0, angle));
            }
        }
    }

    //Keeps an eye on the number of cities left.
    private void GetCities()
    {
        targetCities.Clear();

        GameObject[] cityObjects = GameObject.FindGameObjectsWithTag("Building");

        foreach (GameObject cityObject in cityObjects)
        {
            targetCities.Add(cityObject.transform.position);
        }
    }

    //Returns a random city that is still left.
    public Vector2 TargetCity()
    {
        //Returns the transform of a random city for an enemy missile target
        var target = Random.Range(0, (targetCities.Count));
        var targetCity = new Vector2(targetCities[target].x, targetCities[target].y);
        return targetCity;
    }
```

The above enemy missile manager does most of the heavy lifting. The actual script on the enemy missiles just moves them around and destroys the cities when the missile impacts them.
```
public class EnemyMissile : MonoBehaviour
{
    public Vector3 target;
    public Rigidbody2D rb;
    private float speed = 3000.0f;
    public GameObject explosion;
    private Vector3 currentPos;

    // Update is called once per frame
    void Update()
    {
        //Moves the missile and keeps track of the current pos. If the missile gets to where a city would be, but it was already hit, the missile will destroy itself.
        Vector3 direction = (target - transform.position).normalized;

        rb.velocity = speed * Time.deltaTime * direction;

        currentPos = new Vector3(transform.position.x, transform.position.y);

        if (currentPos.y < -4)
        {
            Destroy(gameObject);
        }
    }

    //If the missile hits a building, makes an explosion effect destroys both the building and itself.
    private void OnTriggerEnter2D (Collider2D collider)
    {
        if (collider.gameObject.tag == "Building")
        {
            Instantiate(explosion, currentPos, Quaternion.identity);
            Destroy(collider.gameObject);
            Destroy(gameObject);
        }
    }
}
```

It was also here that I created my player explosion and its effects. One of the easier scripts to make in this project.
```
public class PlayerExplosion : MonoBehaviour
{
    public float time = 0.7f;
    public AudioSource playerExplosionAudio;

    private void Start()
    {
        playerExplosionAudio.PlayOneShot(playerExplosionAudio.clip, 0.5f);
    }

    // Update is called once per frame
    void Update()
    {
        //Count down to when the explosion goes away
        if (time > 0)
        {
            time -= Time.deltaTime;
        }
        else
        {
            Destroy(gameObject);
        }
    }

    private void OnTriggerStay2D(Collider2D collision)
    {
        //If there is an enemy missile in the explosion, it is destroyed.
        if (collision.gameObject.tag == "Enemy")
        {
            Destroy(collision.gameObject);
        }
    }
}
```

## Story 4: Gameplay
With the base foundational elements needed to make the game work in place, this story was focused on making the actual gameplay.
Everything needed for this was put in a single GameManger script. This covered missile stock piles, loading out of the gameplay scene when all the cities were gone, round advancement, and much more. 
At 114 lines, this is the biggest script in the project, with methods being used by several others.
```
public class MissileGameManager : MonoBehaviour
{
    public SceneLoader loader;
    public List<GameObject> cities = new List<GameObject>();
    public TMP_Text missileCountText;
    public int missileCount = 10;
    private float playerMissileSpawnTime = 10.0f;
    private float roundTimer = 30.0f;
    private int round = 1;

    // Update is called once per frame
    void Update()
    {
        //Updates the various functions. Also keeps an eye on if there are any cities left. If they are all gone, the end scene is loaded
        AdvanceRound();
        AddMissiles();
        missileCountText.text = MissileCountToText(missileCount);
        GetCities();
        if (cities.Count == 0)
        {
            loader.LoadSceneName("Missile-EndScreen");
        }
    }

    //Lowers the missile count each time one is fired
    public void FiredMissile()
    {
        if (missileCount <= 0)
        {
            missileCount = 0;
        }
        else
        {
            missileCount -= 1;
        }
    }

    //Checks to see if there are any missiles left. Returns true or false as needed.
    public bool AreThereMissilesLeft()
    {
        if (missileCount == 0)
        {
            return false;
        }
        else
        {
            return true;
        }
    }

    //Getter for the current round
    public int GetRound()
    {
        return round;
    }

    //Adds missiles every 10 seconds, then resets the timer
    private void AddMissiles()
    {
        if (playerMissileSpawnTime > 0)
        {
            playerMissileSpawnTime -= Time.deltaTime;
        }
        else
        {
            missileCount += 5;
            playerMissileSpawnTime = 10.0f;
        }
    }

    //Keeps track of the current number of cities
    private void GetCities()
    {
        cities.Clear();

        GameObject[] cityObjects = GameObject.FindGameObjectsWithTag("Building");

        foreach (GameObject cityObject in cityObjects)
        {
            cities.Add(cityObject);
        }
    }

    //Casts the int value of missile count to a string
    private string MissileCountToText(int count)
    {
        return count.ToString();
    }

    //Every 30 seconds advances the round by 1, but only if it's not round 6.
    private void AdvanceRound()
    {
        if (roundTimer > 0)
        {
            roundTimer -= Time.deltaTime;
        }
        else if (round != 6)
        {
            round += 1;
            roundTimer = 30.0f;
        }
        else
        {
            round = 6;
            roundTimer = 3000.0f;
        }
    }
}
```

## Story 5: Polish
This finial story was a stright forward polishing run. I added the sound clips you see in the above scripts as well as added the rotation for enemy/player missiles.

## Skills learned
- How to effectivly work by myself. I had help from the instructors if I got stuck, but a lot of the time I had to reserch a problem myself. I did a lot of googling during this project to find help.
- How to develop a game in Unity from start to finish.
- The scrum method. We worked with the scrum method during my time here. Every day we had a standup with a longer meeting on Fridays.
- How to problem solve and refactor code. One of the biggest issues I faced during my work on this was a bug with getting cities that had been destroyed, giving an out of bounds error for the list. The key here was to wipe the list and remake it every Update. Once I figured that out, things went much more smoothly.
