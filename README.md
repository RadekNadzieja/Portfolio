
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
