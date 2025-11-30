
# Portfolio

Some of the code I wrote.


## Some Project

As I can't disclose much, here are 2 fragments of code from the project I'm working on as a freelancer:

Firstly, implementation of netcode for UI purposes:

```
...
class GameplayNetworkView : NetworkBehaviour
    {
        readonly NetworkVariable<int> _timer = new();
        readonly NetworkVariable<int> _totalStars = new();
        NetworkList<int> _spriteIds = new();
        NetworkList<bool> _spriteVisibilities = new();
        NetworkVariable<float> _starbar = new();

        public override void OnNetworkSpawn()
        {
            base.OnNetworkSpawn();
            
            if (NetworkManager.Singleton.IsServer)
                InitializeNetworkLists();

            if (NetworkManager.Singleton.IsClient)
            {
                _timer.OnValueChanged += TimerValueUpdate;
                _totalStars.OnValueChanged += TotalStarsValueUpdate;
                _spriteIds.OnListChanged += SpriteIdsUpdate;
                _spriteVisibilities.OnListChanged += SpriteVisibilitiesUpdate;
                _starbar.OnValueChanged += StarbarValueUpdate;
            }
        }

        public override void OnNetworkDespawn()
        {
            base.OnNetworkDespawn();

            if (NetworkManager.Singleton.IsServer)
            {
                _spriteIds.Clear();
                _spriteVisibilities.Clear();
            }

            if (NetworkManager.Singleton.IsClient)
            {
                _timer.OnValueChanged -= TimerValueUpdate;
                _totalStars.OnValueChanged -= TotalStarsValueUpdate;
                _spriteIds.OnListChanged -= SpriteIdsUpdate;
                _spriteVisibilities.OnListChanged -= SpriteVisibilitiesUpdate;
                _starbar.OnValueChanged -= StarbarValueUpdate;
            }
        }
    ...
```

As there was no need to use RPC at the moment, NetworkVariable is sufficient (and NetworkList).

Example method invoked if there is a change of NetworkVariable value:

```
        ...
        void StarbarValueUpdate(float previousProgress, float currentProgress)
        {
            if (_starbar.Value == currentProgress)
                return;

            Signals.UIStarbarUpdated(currentProgress);
        }
        ...
```
After NetworkVariable update, the subscribed method in invoked. If the value differs from the previous one, Signal is called, and some other script reacts to it by performing an action (in this example - updating the UI element).
"Signals" is inside project architecture system that basically acts as an event system between assemblies.

And the method that acts as a setter for the value of NetworkVariable:

```
        ...
        internal void ChangeStarbar(float newProgress)
        {
            if (!NetworkManager.Singleton.IsHost)
                return;

            Assert.IsTrue(newProgress >= 0f && newProgress <= 1f, "Progress must be in range 0 to 1 inclusive");
            _starbar.Value = newProgress;
            Signals.UIStarbarUpdated(newProgress);
        }
        ...
```
Only host is able to change the value. Assert is used here as a help for debugging.

Secondly, simple Service for changing the position of an object:

```
static class PerfectRoomDisplayService
    {
        static readonly PerfectRoomConfig _config;
        static LevelSceneReferenceHolder _level;
        static Transform _perfectRoomSpawnPoint;
        static Transform _perfectRoomZoomPoint;
        static Transform _cameraTransform;
        static GameObject _perfectRoom;
        static bool _isZoomed;

        internal static void SpawnPerfectRoom(int levelId)
        {
            Assert.IsTrue(levelId >= 0 && levelId < _config.PerfectRooms.Count, "Invalid level ID or PerfectRoomConfig is not populated");
            GameObject prefab = _config.PerfectRooms[levelId].PerfectRoomPrefab;
            _perfectRoom = GameObject.Instantiate(prefab, _perfectRoomSpawnPoint.position, _perfectRoomSpawnPoint.rotation);
            _perfectRoom.transform.SetParent(_cameraTransform, true);
        }

        internal static void ZoomSwitch()
        {
            if (_perfectRoom == null)
                return;
            
            if (_isZoomed)
            {
                _perfectRoom.transform.position = _perfectRoomSpawnPoint.position;
                _perfectRoom.transform.rotation = _perfectRoomSpawnPoint.rotation;
                _isZoomed = false;
                return;
            }
            
            _perfectRoom.transform.position = _perfectRoomZoomPoint.position;
            _perfectRoom.transform.rotation = _perfectRoomZoomPoint.rotation;
            _isZoomed = true;
        }
        
        internal static void Initialize(LevelSceneReferenceHolder level)
        {
            _level = level;
            _perfectRoomSpawnPoint = _level.PerfectRoomSpawnPoint;
            _perfectRoomZoomPoint = _level.PerfectRoomZoomPoint;
            _cameraTransform = _level.Camera;
            _isZoomed = false;
        }
    }
```
Starting with SpawnPerfectRoom() - after the method is invoked by the ViewModel, specific prefab is assigned from config based on levelId. Prefab is then instantiated. After this the ZoomSwitch() is called via input system - the object moves to another position and it stays there as long as the button remains pressed.
## Personal Projects

This is an example of the code from an older project of mine (less cleaner code).

Constructor of the class that holds parameters and contains other methods for various calculations:

```
...
public HexSlot(int _q, int _r, GameObject _prefab, GameObject _decoPrefab, MapGeneratorSettings mapSettings)
    {
        q = _q;
        r = _r;
        float x = 3f / 2 * _q;
        float z = Mathf.Sqrt(3) / 2 * _q + Mathf.Sqrt(3) * _r;
        worldPoint = Vector3.right * x + Vector3.up * 1 + Vector3.forward * z;
        CalculateHeight(mapSettings);
        if (_prefab != null)
        {
            prefab = _prefab;
        }
        else
        {
            prefab = Resources.Load<GameObject>("hexagon");
        }
        if (_decoPrefab != null)
        {
            decoPrefab = _decoPrefab;
        }
    }
...
```
Calculates position of the hexagonal object to fit in the hex grid like pattern. Also calculates Y-position based on the generator settings (Perlin noise, octaves, etc.)

Results of project the code is used in:

![map_big](https://github.com/RadekNadzieja/Portfolio/blob/main/map_big.jpg)

![map2](https://github.com/RadekNadzieja/Portfolio/blob/main/map2.jpg)

Next is still WIP project which also acts as an engineering thesis project for the college.

Firstly, method that finds two closest positions (of the biomes) which are later used for Voronoi Diagram:

```
...
private (int firstBiome, int secondBiome) GetClosestBiomes(Vector3 position)
    {
        Vector3 flatPosition = new Vector3(position.x, 0, position.z);
        int closestBiome = 0;
        int secondClosestBiome = 0;
        float closestDistance = Mathf.Infinity;
        float secondClosestDistance = Mathf.Infinity;

        Vector3 directionToTargetBiome = biomePositions[0] - flatPosition;
        float distance = directionToTargetBiome.magnitude;
        closestDistance = distance;
        
        for (int i = 1; i < biomePositions.Count; i++){
            
            directionToTargetBiome = biomePositions[i] - flatPosition;
            distance = directionToTargetBiome.magnitude;

            if (distance < closestDistance)
            {
                secondClosestDistance = closestDistance;
                secondClosestBiome = closestBiome;
                closestDistance = distance;
                closestBiome = i;
            } else if (distance < secondClosestDistance)
            {
                secondClosestDistance = distance;
                secondClosestBiome = i;
            }
        }
        
        return (closestBiome, secondClosestBiome);
    }
...
```

Secondly, method that calculates arrays of vertex indices for determining triangles of LOD levels of rendered chunks. Arrays are then returned and stored for later access.

Results of project the code is used in:

![map_big](https://github.com/RadekNadzieja/Portfolio/blob/main/map_big.jpg)

![map_big](https://github.com/RadekNadzieja/Portfolio/blob/main/map2.jpg)
