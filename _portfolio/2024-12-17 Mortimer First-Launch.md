---
title:  "Mortimer: First Launch"
date:   2024-12-17 12:00:00 +0300
categories: [JAMK]
tags: [Game, C#, Unity, Steam]
excerpt: "A whimsical stealth-puzzle adventure game starring Mortimer, a curious raccoon 🦝"
#header:
#  image: /assets/images/MortimerHeader.jpg
#  teaser: assets/images/unsplash-gallery-image-1-th.jpg
sidebar:
  - title: "Role"
    text: "Lead programmer"
  - title: "Responsibilities"
    text: "Debugging, Q&A Lead, Implementing gameplay mechanics"
gallery:
  - url: /assets/images/MortimerHeader.jpg
    image_path: assets/images/MortimerHeader.jpg
    alt: "Header"
  - url: /assets/images/Mortimer_ss1.jpg
    image_path: assets/images/Mortimer_ss1.jpg
    alt: "Screenshot 1"
  - url: /assets/images/Mortimer_ss2.jpg
    image_path: assets/images/Mortimer_ss1.jpg
    alt: "Screenshot 2"
order: 1
---

<iframe src="https://store.steampowered.com/widget/3288660/?utm_source=portfoliopage" width="100%" height="200"></iframe>

This project page was created as part of my university studies and follows the guidelines set by the institution. As such, it contains content, such as academic descriptions, and details that may not align with how I would normally present projects in my portfolio. Certain sections may feel overly detailed or irrelevant in the context of a professional portfolio, but they are included to meet academic requirements.

---

# Programming
I was Lead Programmer on **Mortimer: First Launch** project, I implemented most of the gameplay logic.
Here is some code snippets to show what have I done

## Surface Detector
Surface detector is part of the audio system, it changes footsteps audio depending what kind of surface player is walking on. Making this was quite straight forward, only problem was to find out how Unity stores terrain layer weights to the terrain data.

{% highlight c# %}
namespace MortimerDreamsOfSpace.WwiseIntegration
{
    public class SurfaceDetector : MonoBehaviour
    {
        [SerializeField] private LayerMask _groundLayerMask;
        [SerializeField] private List<PhysicsMaterialMapping> physicsMaterialMappings;
        [SerializeField] internal UnityEvent<SurfaceChangeEventArgs> onSurfaceChange;
        [SerializeField] private SurfaceChangeEventArgs _previousArgs;

        void FixedUpdate()
        {
	        // Raycasting down
            if (Physics.Raycast(transform.position, Vector3.down, out RaycastHit hit, 1f))
            {
                // Check layer mask
                if ((hit.collider.gameObject.layer | _groundLayerMask) == 0)
                    return;
                var args = new SurfaceChangeEventArgs();

                // Check if the surface has been manually assigned to the object
                if (hit.collider.TryGetComponent(out Surface surface))
                {
                    args.AudioId = surface.AudioId;
                    args.Surface = surface;
                }
                // Checking is collider terrain and getting related terrain audio mapping
                else if (hit.collider.TryGetComponent(out Terrain terrain) && hit.collider.TryGetComponent(out TerrainAudioMapping audioMapping))
                {
                    var layer = GetDominantTextureIndex(terrain, hit.point);
                    if (layer < 0) // Terrain detection bug, skipping detection this time
                        return;

                    args.AudioId = audioMapping.GetAudioId(layer);
                }
                // Trying to solve audio for the physics material
                else if (hit.collider.sharedMaterial != null)
                {
                    var surfaceMaterial = physicsMaterialMappings.Find(o => o.physicMaterial == hit.collider.sharedMaterial);
                    if (surfaceMaterial == null)
                        return;
                    args.AudioId = surfaceMaterial.audioId;
                }

				// If previous is same as current, do nothing
                if (_previousArgs != null && args.Equals(_previousArgs))
                    return;

				// Changing audio material
                Debug.Log("Surface changed to " + args);
                _previousArgs = args;
                onSurfaceChange?.Invoke(args);
            }
        }

        public int GetDominantTextureIndex(Terrain terrain, Vector3 worldPos)
        {
            TerrainData terrainData = terrain.terrainData;

            // Convert world coordinates to terrain coordinates
            Vector3 terrainPos = worldPos - terrain.transform.position;
            float xCoord = terrainPos.x / terrainData.size.x; // On scale of 0 to 1
            float zCoord = terrainPos.z / terrainData.size.z;

            // Get the alpha map position
            int mapX = Mathf.RoundToInt(xCoord * terrainData.alphamapWidth);
            int mapZ = Mathf.RoundToInt(zCoord * terrainData.alphamapHeight);

            // BUG: Something causing a bug that causes negative coordinates
            if (mapX < 0 || mapZ < 0)
                return -1;

            // Get the alpha map (splat map)
            float[,,] alphaMap = terrainData.GetAlphamaps(mapX, mapZ, 1, 1);

            // Find the dominant texture index
            int dominantTextureIndex = 0;


            for (int i = 0; i < alphaMap.GetLength(2); i++)
            {
                if (alphaMap[0, 0, i] > 0.05f)
                {
                    dominantTextureIndex = i;
                }
            }

            return dominantTextureIndex;
        }

        [System.Serializable]
        public class SurfaceChangeEventArgs : System.IEquatable<SurfaceChangeEventArgs>
        {
            public AudioId AudioId { get; internal set; }
            public Surface Surface { get; internal set; }

            public bool Equals(SurfaceChangeEventArgs other)
            {
                return other.Surface == Surface && other.AudioId == AudioId;
            }

            public override string ToString()
            {
                string txt = string.Empty;
                if (Surface == null)
                    txt += "nullSurface";
                else
                    txt += Surface.name;

                if (AudioId == null)
                    txt += " nullAudioId";
                else
                    txt += " " + AudioId.name;

                return txt;
            }
        }
    }
}
{% endhighlight %}

## Achievements
Game uses Steam Achievements, this was first time when implementing achievements system. I made this part quickly and it doesn't cache current status anyhow but just sets the achievement received every time when it is triggered. This is using Unitys ScriptableObjects, so achievement can be triggered directly with UnityEvents. This is using Steamworks.NET C# Wrapper (com.rlabrecque.steamworks.net)

![Screenshot 1](/assets/images/Mortimer_pss1.png)

{% highlight c# %}
namespace MortimerDreamsOfSpace.Achievement
{
    [CreateAssetMenu(fileName = "Achievement", menuName = "MortimerDreamsOfSpace/Achievement")]
    public class Achievement : ScriptableObject
    {
        [SerializeField] private string apiName;
        [SerializeField] string achievementName;
        public void Unlock()
        {
            Debug.Log("Unlocking achievement: " + achievementName);
            if(!SteamManager.Initialized) return;
            SteamUserStats.SetAchievement(apiName);
            SteamUserStats.StoreStats();
        }

        public void Lock()
        {
            Debug.Log("Locking achievement: " + achievementName);
            if (!SteamManager.Initialized) return;
            SteamUserStats.ClearAchievement(apiName);
            SteamUserStats.StoreStats();
        }

        static string ConvertToUpperWithUnderscores(string input)
        {
            string result = Regex.Replace(input, "(?<!^)([A-Z])", "_$1");
            return result.ToUpper();
        }

#if UNITY_EDITOR
        private void OnValidate()
        {
            if (string.IsNullOrEmpty(apiName))
            {
                apiName = "ACH_" + ConvertToUpperWithUnderscores(name);
            }
        }
#endif
    }
}
{% endhighlight %}

## NPC Ai
Ai brain is the core of the NPC AI system, it switches the AI state from request and if game object has multiple states that matches the request, it picks random state that satisfies the request. AI states are built hierarchy using inheritance, all the AI states base class is AiState, and if Brain gets a request to 

for example, if we have following inheritance hierarchy and WorkingState is requested, it pick pick CookingState, HuntingState or PatrollState. 
![Ai diagran](/assets/images/Mortimer_AiDiagram.png)

{% highlight c# %}
namespace MortimerDreamsOfSpace.AI
{
    [SelectionBase]
    public class AiBrain : MonoBehaviour
    {
        [SerializeField] public NpcSettings settings;
        [SerializeField] public UnityEvent TargetReachedEvent;
        [SerializeField] public UnityEvent<NpcThoughtBubbleContent> onThought;
        [SerializeField, Range(0f, 1f)] internal float movementRate = 0f;

        internal Transform destination;

        public Vector3 HomePosition { get; private set; }

        private protected AiState[] _allStates;
        private AiTaskManager[] taskManager;
        private AiState _currentState;
        private AiState _previousState;
        private int _previousAnimationState;
        HashSet<Collider> _activatorColliders = new();

        private List<AiTrigger> _activeTriggers = new();
        private bool paused;

        public Type CurrentStateType { get => _currentState ? _currentState.GetType() : null; }
        public bool TargetReached { get; private set; }
        public float Speed { get; private set; }

        // Awake is called when the script instance is being loaded
        private void Awake()
        {
            settings.agent = GetComponent<NavMeshAgent>();
            settings.cc = GetComponent<CharacterController>();
            settings.aiSensor = GetComponentInChildren<AiSensor>();
            settings.animator = GetComponent<Animator>();
            settings.headBone = settings.animator.GetBoneTransform(HumanBodyBones.Head);
            settings.aiSensor.SetSize(MathF.Max(settings.ViewDistance, settings.HearDistance));
            settings.aiSensor.onSensorEnter.AddListener(OnAiTriggerEnter);
            settings.aiSensor.onSensorExit.AddListener(OnAiTriggerExit);

            // Storing home position so AI can return to it
            HomePosition = transform.position;

            // Initializing states
            _allStates = GetComponents<AiState>();
            _currentState = _previousState = _allStates.Where(s => s.enabled && s.GetType() == typeof(IdleState)).FirstOrDefault();

            foreach (var state in _allStates)
                state.Initialize(this, settings);

            // Setting initial state
            ChangeState(typeof(IdleState));
        }

        internal virtual void Update()
        {
            if (paused)
                return;

            UpdateAnimator();

            if (_currentState != null)
                _currentState.UpdateState();
        }

        /// <summary>
        /// Request AI to change state
        /// </summary>
        /// <param name="newStateType">Target state (or random child type)</param>
        public void ChangeState(Type newStateType)
        {
            // Getting matching states including inherited types
            var matchingStates = _allStates.Where(s => newStateType.IsAssignableFrom(s.GetType())).ToList();
            var newState = matchingStates.Count > 0 ? matchingStates[UnityEngine.Random.Range(0, matchingStates.Count)] : null;

            if (newState == null)
            {
                Debug.LogWarning($"No state found for type {newStateType}", gameObject);
                return;
            }

            // If new state is the same as the current state, do nothing
            if (newState == _currentState)
                return;

            // Exiting previous state
            if (_currentState != null)
                _currentState.ExitState();
            _currentState = newState;

            // Sending current detections to new state
            _currentState.OnDetectionChange(_activeTriggers);

            // Entering new state
            if (_currentState != null)
                _currentState.EnterState();
        }

        /// <summary>
        /// Sets AI destination
        /// </summary>
        /// <param name="destination"></param>
        internal void SetDestination(Vector3 destination, float speed = 1f)
        {
            Speed = speed;
            if (settings.agent == null)
                return;
            this.destination = null;
            TargetReached = false;
            settings.agent.SetDestination(destination);
        }

        /// <summary>
        /// Sets or reset AI destination
        /// </summary>
        /// <param name="transform"></param>
        internal void SetDestination(Transform transform = null, float speed = 1f)
        {
            Speed = speed;
            if (settings.agent == null)
                return;
            if (transform != null)
            {
                destination = transform;
                TargetReached = false;
                settings.agent.SetDestination(destination.position);
            }
            else
                settings.agent.isStopped = true;
        }

		// When something enters to the AI detection zone
        private void OnAiTriggerEnter(AiTrigger arg0)
        {
            if (_activeTriggers.Contains(arg0))
                return;
            _activeTriggers.Add(arg0);
            _currentState.OnDetectionChange(_activeTriggers);
        }
        private void OnAiTriggerExit(AiTrigger arg0)
        {
            _activeTriggers.Remove(arg0);
            _currentState.OnDetectionChange(_activeTriggers);
        }

        private void UpdateAnimator()
        {
            if (settings.agent.hasPath)
            {
                Vector3 velocity = settings.agent.desiredVelocity;
                Vector3 localVelocity = transform.InverseTransformDirection(velocity).normalized;
                var dir = (settings.agent.steeringTarget - transform.position).normalized;
                var facingToMoveDirection = Vector3.Dot(transform.forward, dir) > .5f;

                settings.animator.SetFloat(AnimatorID.Horizontal, facingToMoveDirection ? localVelocity.x * Speed : 0f, .5f, Time.deltaTime);
                settings.animator.SetFloat(AnimatorID.Vertical, localVelocity.z * Speed, .5f, Time.deltaTime);

                if (settings.agent.remainingDistance < settings.agent.radius + settings.agentStopDistance)
                {
                    settings.agent.ResetPath();
                    TargetReached = true;
                    TargetReachedEvent?.Invoke();
                    SetAnimationState(0);
                    _currentState.OnDestinationReached();
                }
                else
                {
                    SetAnimationState(1); // Walking/Running
                }
            }
            else
            {
                settings.animator.SetFloat(AnimatorID.Horizontal, 0f, .25f, Time.deltaTime);
                settings.animator.SetFloat(AnimatorID.Vertical, 0f, .25f, Time.deltaTime);
                SetAnimationState(0); // Idle
            }
        }

		// Setting Animator state
        private void SetAnimationState(int state)
        {
            if (state == _previousAnimationState)
                return;

            settings.animator.SetInteger(AnimatorID.LastState, _previousAnimationState);
            settings.animator.SetInteger(AnimatorID.State, state);
            settings.animator.SetTrigger(AnimatorID.StateOn);
            _previousAnimationState = state;
        }

		// Checking can AI see the target
        internal bool IsVisible(AiTrigger target)
        {
            if (Vector3.Angle(settings.headBone.forward, target.transform.position - settings.headBone.position) > settings.Fov / 2)
                return false;

            foreach (Collider c in target.Colliders)
            {
                Ray r = new Ray(settings.headBone.position, (c.gameObject.transform.position - settings.headBone.position));
                if (Physics.Raycast(r, out RaycastHit hit, settings.ViewDistance * target.VisualDetectionProbability, settings.aiSensor.layerMask))
                {
                    if (hit.collider.GetComponentInParent<AiTrigger>() == target)
                    {
                        Debug.DrawRay(r.origin, r.direction * settings.ViewDistance, Color.green, 1f);
                        return true;
                    }
                }
                else
                {
                    Debug.DrawRay(r.origin, r.direction * settings.ViewDistance, Color.red, 1f);
                }
            }
            return false;
        }

		// Can AI hear the target
        internal bool IsHearing(AiTrigger target)
        {
            if (Vector3.Distance(settings.headBone.position, target.transform.position) < settings.HearDistance * target.AudioDetectionProbability)
            {
                Debug.DrawLine(settings.headBone.position, target.transform.position, Color.green, 1f);
                return true;
            }
            return false;
        }

		// Pausing the AI if game is paused
        public void PauseAI(bool pause)
        {
            paused = pause;
            if (settings.agent == null)
                return;
            settings.agent.isStopped = pause;
        }

#if UNITY_EDITOR
        private void OnDrawGizmosSelected()
        {
            settings.OnDrawGizmos(transform);
            if (_currentState != null)
                UnityEditor.Handles.Label(settings.headBone.position, _currentState.GetType().Name);
        }

        private void OnDrawGizmos()
        {
            if (settings.agent == null)
                return;

            for (int i = 0; i < settings.agent.path.corners.Length - 1; i++)
            {
                Debug.DrawLine(settings.agent.path.corners[i], settings.agent.path.corners[i + 1], Color.blue);
            }
            Gizmos.DrawWireSphere(settings.agent.destination, settings.agentStopDistance);
        }
#endif
    }
}
{% endhighlight %}

## Custom build script
For this project I created a custom build script that builds Windows and Linux builds for Steam and creates .vdf file for steam upload to steamline the development and testing process. Following script has been improved later on to be suitable to other project too, but this is the version that we was using on this project.

{% highlight c# %}
public class SteamBuildScript
{
    private const string steamBuildsPath = "SteamBuilds";
    private static string targetBinary;
    private static string gitString;

    [MenuItem("Mortimer/SteamBuild", priority = 30)]
    public static void BuildSteam()
    {
        CheckDependencies(); // This will throw an exception if dependencies are not met

        gitString = GenerateDescription();
        string appBuildVdf = Path.Combine(steamBuildsPath, "app_build_3288660.vdf");
        CreateAppVdfFile(appBuildVdf);

        bool success = true;
        if (success)
            success &= Build(BuildTarget.StandaloneWindows64);

        if (success)
            success &= Build(BuildTarget.StandaloneLinux64);

        if (!success)
            Debug.LogError("Build failed");

        EditorUtility.RevealInFinder(Path.Combine(steamBuildsPath, "UploadToSteam.bat"));

        // Cleanup
        if (File.Exists("Assets/Wwise"))
        {
            RunGit("restore Assets/Wwise");
        }
    }

    [MenuItem("Mortimer/Open SteamBuild Folder", priority = 31)]
    private static void OpenFolder()
    {
        EditorUtility.RevealInFinder(Path.Combine(steamBuildsPath, ""));
    }

    private static string GenerateDescription()
    {
        try
        {
            return RunGit("describe --dirty --tags");
        }
        catch (Exception e)
        {
            Debug.LogError("Failed to generate build description: " + e.Message);
            return "Unknown";
        }
    }

    private static void CheckDependencies()
    {
        // Is git installed?
        Debug.Log(RunGit("-v"));

        // Is Toolchain Win Linux x64 installed?
        ListRequest listRequest = Client.List(true);
        while (!listRequest.IsCompleted)
        {
            var cancelled = EditorUtility.DisplayCancelableProgressBar("Checking dependencies", "Checking if Toolchain Win Linux x64 is installed", 0.5f);
            if (cancelled)
                break;
        }
        EditorUtility.ClearProgressBar();

        if (listRequest.Status == StatusCode.Success)
        {
            if (!listRequest.Result.Any(package => package.name == "com.unity.toolchain.win-x86_64-linux-x86_64"))
                throw new Exception("Toolchain Win Linux x64 is not installed");
        }
        else
        {
            // Log an error if the request failed
            throw new Exception("Failed to list packages: " + listRequest.Error?.message);
        }

        // Is Steamworks SDK installed
        if (!File.Exists(Path.Combine(steamBuildsPath, "sdk", "tools", "ContentBuilder", "builder", "steamcmd.exe")))
            throw new Exception("Steamworks SDK is not installed");
    }

    private static void CreateAppVdfFile(string appBuildVdf)
    {
        if (string.IsNullOrWhiteSpace(gitString))
            throw new Exception("Description is empty");

        var app = new AppBuildVdf("3288660");
        app.Desc = gitString;
        app.SetLive = "AlphaTest";
        //app.Preview = false;
        app.ContentRoot = ".";
        app.BuildOutput = ".\\build_output\\";
        app.Depots.Add("3288661", "depot_build_3288661_windows.vdf");
        app.Depots.Add("3288662", "depot_build_3288662_linux.vdf");
        app.Save(appBuildVdf);
    }

    public static bool Build(BuildTarget targetPlatform)
    {
        string targetName = Application.productName.Replace(":", "");
        if (targetPlatform == BuildTarget.StandaloneWindows64 || targetPlatform == BuildTarget.StandaloneWindows)
            targetBinary = targetName + ".exe";
        if (targetPlatform == BuildTarget.StandaloneLinux64)
            targetBinary = targetName;

        // Generate date string
        string date = DateTime.Now.ToString("yyyy-MM-dd_HH-mm");

        // Define the full path for the build folder
        string buildPath = Path.Combine(steamBuildsPath, $"Build-" + targetPlatform.ToString());

        // Create the directory if it doesn’t exist
        Directory.CreateDirectory(buildPath);

        // Creating log folder
        string buildLogPath = Path.Combine(steamBuildsPath, "build_output");
        Directory.CreateDirectory(buildLogPath);

        // Build the project
        BuildReport result = BuildPipeline.BuildPlayer(EditorBuildSettings.scenes, Path.Combine(buildPath, targetBinary), targetPlatform, BuildOptions.None);

        // Check if the build succeeded and output the result
        if (result.summary.result == BuildResult.Succeeded)
        {
            var sb = new StringBuilder();
            sb.AppendLine("Build result   : " + result.summary.result);
            sb.AppendLine("Build size     : " + result.summary.totalSize + " bytes");
            sb.AppendLine("Build time     : " + result.summary.totalTime);
            //sb.AppendLine("Error summary  : " + result.SummarizeErrors());
            sb.Append(LogBuildReportSteps(result));
            sb.AppendLine(LogBuildMessages(result));

            // Output log message
            File.WriteAllText(Path.Combine(buildLogPath, date + "-" + gitString + "-" + targetPlatform.ToString() + ".log"), sb.ToString());
            Debug.Log($"Build completed: {buildPath}");
            return true;
        }
        else
        {
            var sb = new StringBuilder();
            sb.AppendLine("Build result   : " + result.summary.result);
            sb.AppendLine("Build size     : " + result.summary.totalSize + " bytes");
            sb.AppendLine("Build time     : " + result.summary.totalTime);
            //sb.AppendLine("Error summary  : " + result.SummarizeErrors());
            sb.Append(LogBuildReportSteps(result));
            sb.AppendLine(LogBuildMessages(result));

            // Output log message
            File.WriteAllText(Path.Combine(buildLogPath, date + "-" + gitString + "-" + targetPlatform.ToString() + "-fail.log"), sb.ToString());
            Debug.LogError($"Build failed: {result.summary.result}");
            return false;
        }
    }

    public static string LogBuildReportSteps(BuildReport buildReport)
    {
        var sb = new StringBuilder();

        sb.AppendLine($"Build steps: {buildReport.steps.Length}");
        int maxWidth = buildReport.steps.Max(s => s.name.Length + s.depth) + 3;
        foreach (var step in buildReport.steps)
        {
            string rawStepOutput = new string('-', step.depth + 1) + ' ' + step.name;
            sb.AppendLine($"{rawStepOutput.PadRight(maxWidth)}: {step.duration:g}");
        }
        return sb.ToString();
    }

    public static string LogBuildMessages(BuildReport buildReport)
    {
        var sb = new StringBuilder();
        foreach (var step in buildReport.steps)
        {
            foreach (var message in step.messages)
                sb.AppendLine($"[{message.type}] {message.content}");
        }

        string messages = sb.ToString();
        if (messages.Length > 0)
            return "Messages logged during Build:\n" + messages;
        else
            return "";
    }

    private static string RunGit(string args)
    {
        System.Diagnostics.ProcessStartInfo psi = new System.Diagnostics.ProcessStartInfo
        {
            FileName = "git",
            Arguments = args,
            RedirectStandardOutput = true,
            RedirectStandardError = true,
            UseShellExecute = false,
            CreateNoWindow = true,
            WindowStyle = System.Diagnostics.ProcessWindowStyle.Hidden
        };

        using (System.Diagnostics.Process process = new System.Diagnostics.Process())
        {
            process.StartInfo = psi;
            process.Start();

            // Capture output
            string output = process.StandardOutput.ReadToEnd();
            string error = process.StandardError.ReadToEnd();

            process.WaitForExit();

            if (!string.IsNullOrEmpty(error))
            {
                throw new Exception($"Git error: {error}");
            }

            return output.Trim();
        }
    }
}
{% endhighlight %}

# Debugging
We used new tools that whole team was unfamiliar with, and documentation was poor, so my job was also find out how things works and whenever there was a code side problems, it was my job to find out what is wrong and fix them.

# Implementation
On this project I bult most of the complex prefab setups with logic like NPCs, Conveyer belt, Menus

![Screenshot 2](/assets/images/Mortimer_pss2.png)

{% include gallery caption="Project gallery" %}

