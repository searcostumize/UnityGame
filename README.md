# Unity 3D WebGL Game

Dit is een 3D Unity-game die geüpload is naar GitHub Pages. Je kunt de game spelen in je browser via de onderstaande link.

## Speel de game

Klik op de onderstaande link om de game te spelen:

[(https://jouwgebruikersnaam.github.io/unity-webgl-game/](https://searcostumize.github.io/UnityGame/))

## Beschrijving

Deze game is gemaakt als onderdeel van een schoolproject en is gebouwd in Unity. De game maakt gebruik van verschillende Unity-functies, zoals 3D-modellering, fysica en gebruikersinteractie. 

## Besturing

- Gebruik de pijltjestoetsen om te bewegen.
- Gebruik de spatiebalk om te springen.

## Code

using UnityEngine;
using UnityEngine.UI;

public class PlayerController : MonoBehaviour
{
    [Header("Movement")]
    public float moveSpeed = 5f;
    public float rotationSpeed = 10f;
    public float gravity = 9.8f;
    private Vector3 velocity;
    public CharacterController controller;
    public Transform cameraTransform;

    [Header("Health & UI")]
    public int health = 100;
    public Slider healthBar;
    public GameObject deathScreen;

    [Header("Damage")]
    private float damageCooldown = 1f;
    private float lastDamageTime = 0f;

    [Header("XP & Evolution")]
    public int xp = 0;
    public Mesh normalMesh;
    public Mesh evolvedMesh;
    public MeshFilter playerMeshFilter;
    public Renderer playerRenderer;
    public Material normalMaterial;
    public Material evolvedMaterial;

    [Header("Consumables")]
    public float consumableSpeed = 3f;
    public float consumableDetectionRange = 5f;

    void Start()
    {
        if (playerMeshFilter && normalMesh)
            playerMeshFilter.mesh = normalMesh;

        if (playerRenderer && normalMaterial)
            playerRenderer.material = normalMaterial;

        if (deathScreen)
            deathScreen.SetActive(false);
    }

    void Update()
    {
        MovePlayer();
        RotateCamera();
        ApplyGravity();
        UpdateHealthUI();
        MoveConsumablesAway();
        ApplyConsumablePhysics();
        CheckXPForEvolution();
    }

    void MovePlayer()
    {
        float moveX = Input.GetAxis("Horizontal");
        float moveZ = Input.GetAxis("Vertical");
        Vector3 move = transform.right * moveX + transform.forward * moveZ;
        controller.Move(move * moveSpeed * Time.deltaTime);
    }

    void RotateCamera()
    {
        float mouseX = Input.GetAxis("Mouse X") * rotationSpeed;
        transform.Rotate(Vector3.up * mouseX);
        cameraTransform.position = transform.position - transform.forward * 3 + Vector3.up * 1.5f;
        cameraTransform.LookAt(transform);
    }

    void ApplyGravity()
    {
        if (!controller.isGrounded)
            velocity.y -= gravity * Time.deltaTime;
        else
            velocity.y = -2f;

        controller.Move(velocity * Time.deltaTime);
    }

    void UpdateHealthUI()
    {
        if (healthBar)
            healthBar.value = health;

        if (health <= 0 && deathScreen)
            deathScreen.SetActive(true);
    }

    void OnTriggerEnter(Collider other)
    {
        GameObject parent = other.CompareTag("ConsumableTrigger") ? other.transform.parent.gameObject : other.gameObject;

        if (parent.CompareTag("Damage"))
        {
            lastDamageTime = Time.time;
        }
        else if (parent.CompareTag("Consumable"))
        {
            HealPlayer(parent);
            GainXP();
        }
        else if (parent.CompareTag("AdvancedConsumable"))
        {
            if (xp >= 5)
            {
                HealPlayer(parent);
            }
            else
            {
                lastDamageTime = Time.time;
            }
        }
    }

    void OnTriggerStay(Collider other)
    {
        GameObject parent = other.CompareTag("ConsumableTrigger") ? other.transform.parent.gameObject : other.gameObject;

        if (parent.CompareTag("Damage") || (parent.CompareTag("AdvancedConsumable") && xp < 5))
        {
            if (Time.time - lastDamageTime >= damageCooldown)
            {
                health -= 5;
                lastDamageTime = Time.time;
            }
        }
    }

    void HealPlayer(GameObject consumable)
    {
        health = Mathf.Min(health + 10, 100);
        Destroy(consumable);
    }

    void GainXP()
    {
        xp += 1;
    }

    void CheckXPForEvolution()
    {
        if (xp >= 5)
        {
            if (playerMeshFilter && evolvedMesh && playerMeshFilter.mesh != evolvedMesh)
            {
                playerMeshFilter.mesh = evolvedMesh;
            }

            if (playerRenderer && evolvedMaterial && playerRenderer.material != evolvedMaterial)
            {
                playerRenderer.material = evolvedMaterial;
            }
        }
    }

    void MoveConsumablesAway()
    {
        GameObject[] consumables = GameObject.FindGameObjectsWithTag("Consumable");
        GameObject[] advanced = GameObject.FindGameObjectsWithTag("AdvancedConsumable");

        foreach (GameObject consumable in consumables)
        {
            float distance = Vector3.Distance(transform.position, consumable.transform.position);
            Rigidbody rb = consumable.GetComponent<Rigidbody>();

            if (rb != null)
            {
                if (distance < consumableDetectionRange)
                {
                    Vector3 dir = (consumable.transform.position - transform.position).normalized;
                    consumable.transform.rotation = Quaternion.LookRotation(dir);
                    rb.velocity = new Vector3(dir.x * consumableSpeed, 0f, dir.z * consumableSpeed);
                }
                else
                {
                    rb.velocity = Vector3.zero;
                }
            }
        }

        foreach (GameObject adv in advanced)
        {
            Rigidbody rb = adv.GetComponent<Rigidbody>();
            if (rb == null) continue;

            if (xp >= 5)
            {
                float dist = Vector3.Distance(transform.position, adv.transform.position);
                if (dist < consumableDetectionRange)
                {
                    Vector3 dir = (adv.transform.position - transform.position).normalized;
                    adv.transform.rotation = Quaternion.LookRotation(dir);
                    rb.velocity = new Vector3(dir.x * consumableSpeed, 0f, dir.z * consumableSpeed);
                }
                else
                {
                    rb.velocity = Vector3.zero;
                }
            }
            else
            {
                rb.velocity = Vector3.zero;
            }
        }
    }

    void ApplyConsumablePhysics()
    {
        GameObject[] allConsumables = GameObject.FindGameObjectsWithTag("Consumable");
        GameObject[] allAdvanced = GameObject.FindGameObjectsWithTag("AdvancedConsumable");

        foreach (GameObject obj in allConsumables)
            SetupConsumable(obj, "ConsumableTrigger");

        foreach (GameObject obj in allAdvanced)
            SetupConsumable(obj, "ConsumableTrigger");
    }

    void SetupConsumable(GameObject consumable, string triggerTag)
    {
        Rigidbody rb = consumable.GetComponent<Rigidbody>();
        if (rb == null)
        {
            rb = consumable.AddComponent<Rigidbody>();
        }

        rb.useGravity = false;
        rb.freezeRotation = true;
        rb.constraints = RigidbodyConstraints.FreezePositionY;

        Collider solidCol = consumable.GetComponent<Collider>();
        if (solidCol == null)
        {
            solidCol = consumable.AddComponent<BoxCollider>();
        }
        solidCol.isTrigger = false;

        if (consumable.transform.Find("Trigger") == null)
        {
            GameObject triggerObj = new GameObject("Trigger");
            triggerObj.transform.parent = consumable.transform;
            triggerObj.transform.localPosition = Vector3.zero;

            SphereCollider triggerCol = triggerObj.AddComponent<SphereCollider>();
            triggerCol.isTrigger = true;
            triggerCol.radius = 1f;

            triggerObj.tag = triggerTag;
        }
    }
}

## Vereisten

- Geen installatie nodig! De game kan direct in de browser gespeeld worden.
- Bekijk de code in de `Assets/Scripts` map voor meer details.

## Credits

- **Ontwikkelaar:** [Jouw Naam]
- **Platform:** Unity 3D
- **Technologie:** WebGL

---

