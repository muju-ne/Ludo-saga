# Ludo-saga
Play with your family and friends 
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.UI;

// GameManager - pass-and-play local implementation with capture & home-stretch exact finish
public class GameManager : MonoBehaviour
{
    [Header("References")]
    public PathManager pathManager;
    public DiceManager diceManager;
    public Text infoText;

    [Header("Players and tokens")]
    public List<Player> players = new List<Player>(); // assign players in inspector (4 players)
    public int currentPlayerIndex = 0;

    private bool isRolling = false;
    private bool isMoving = false;

    void Start()
    {
        if (players == null || players.Count == 0)
        {
            Debug.LogWarning("No players set in GameManager.");
        }

        UpdateUI();
    }

    #region UI hooks
    // Called by UI button
    public void OnRollClicked()
    {
        if (isRolling || isMoving) return;
        isRolling = true;
        diceManager.Roll(OnRolled);
    }
    #endregion

    private void OnRolled(int value)
    {
        isRolling = false;
        StartCoroutine(HandleRoll(value));
    }

    private IEnumerator HandleRoll(int rollValue)
    {
        isMoving = true;
        var player = players[currentPlayerIndex];
        infoText.text = $"Player {currentPlayerIndex + 1} rolled {rollValue}";

        // find movable tokens for this player (prefer tokens on path, else home if 6)
        Token chosen = null;

        // first prefer tokens that are on path or homePath and can move (without overshoot)
        foreach (var t in player.tokens)
        {
            if (CanTokenMove(t, rollValue)) { chosen = t; break; }
        }

        if (chosen == null)
        {
            // no moves available -> skip (unless rule variant gives extra roll on 6, but here we follow classic: if 6 but no token can move, still extra? we'll keep: extra if could leave home)
            infoText.text = "No valid moves. Turn passes.";
            yield return new WaitForSeconds(1f);
            NextTurn();
            isMoving = false;
            yield break;
        }

        // Compute path positions to animate and apply final state
        var positions = new List<Vector3>();

        // If token at home and rollValue==6 -> move to entry main index
        if (chosen.location == Token.LocationType.HomeArea)
        {
            if (rollValue == 6)
            {
                int entryIndex = player.startMainIndex;
                chosen.mainIndex = entryIndex;
                chosen.location = Token.LocationType.MainPath;
                positions.Add(pathManager.GetMainPoint(chosen.mainIndex));
                // apply capture if someone is on this main point
                yield return StartCoroutine(AnimateAndResolve(chosen, positions.ToArray()));
                // Staying on main - extra turn on 6
                if (rollValue == 6) { infoText.text = "Rolled 6 — play again."; isMoving = false; yield break; }
                else { NextTurn(); isMoving = false; yield break; }
            }
            else
            {
                // shouldn't happen, because CanTokenMove already checked roll==6 for home tokens
                infoText.text = "Cannot leave home without a 6.";
                NextTurn();
                isMoving = false;
                yield break;
            }
        }

        // Token is on MainPath or HomePath
        if (chosen.location == Token.LocationType.MainPath)
        {
            int stepsRemaining = rollValue;
            int mainLen = pathManager.MainPathLength;
            int idx = chosen.mainIndex;

            // compute how many steps until this player's main entry to home-stretch
            int homeEntryIndex = player.homeEntryIndex; // index on main path where this player's home path starts
            int stepsToHomeEntry = (homeEntryIndex - idx + mainLen) % mainLen;
            if (stepsToHomeEntry == 0) stepsToHomeEntry = mainLen; // if currently on entry, need mainLen steps to loop — but logically treat as 0

            // If steps move us into home path
            if (stepsRemaining > stepsToHomeEntry)
            {
                // move to home entry first
                int moveOnMain = stepsToHomeEntry;
                for (int s = 0; s < moveOnMain; s++)
                {
                    idx = (idx + 1) % mainLen;
                    positions.Add(pathManager.GetMainPoint(idx));
                }

                int intoHome = stepsRemaining - moveOnMain; // how many steps inside home path
                int homeLen = pathManager.GetHomeLength(chosen.ownerPlayerId);

                // compute proposed homeIndex
                int targetHomeIndex = chosen.homeIndex; // if already -1, entry will be 0
                if (chosen.homeIndex < 0) targetHomeIndex = 0 + (intoHome - 1);
                else targetHomeIndex = chosen.homeIndex + intoHome;

                // exact finish check: cannot overshoot homeLen - 1
                if (targetHomeIndex >= homeLen)
                {
                    // overshoot -> invalid move (token cannot move). For simplicity, we treat as cannot move.
                    infoText.text = "Move would overshoot finish. Move not allowed.";
                    yield return new WaitForSeconds(1f);
                    NextTurn();
                    isMoving = false;
                    yield break;
                }

                // animate onto main then home path points
                // update chosen state progressively
                chosen.mainIndex = idx;
                chosen.location = Token.LocationType.MainPath;
                // add remaining home path positions
                for (int h = 0; h < intoHome; h++)
                {
                    int homePosIndex = (chosen.homeIndex < 0 ? 0 : chosen.homeIndex + 1) + h;
                    positions.Add(pathManager.GetHomePoint(chosen.ownerPlayerId, homePosIndex));
                }

                // after movement update state to home path and homeIndex
                yield return StartCoroutine(AnimateAndResolve(chosen, positions.ToArray(), movedToHome: true, newHomeIdx: (chosen.homeIndex < 0 ? 0 : chosen.homeIndex + intoHome)));
            }
            else
            {
                // purely on main path
                for (int s = 0; s < stepsRemaining; s++)
                {
                    idx = (idx + 1) % mainLen;
                    positions.Add(pathManager.GetMainPoint(idx));
                }
                chosen.mainIndex = idx;
                yield return StartCoroutine(AnimateAndResolve(chosen, positions.ToArray()));
            }
        }
        else if (chosen.location == Token.LocationType.HomePath)
        {
            // already in home path: must not overshoot final spot
            int homeLen = pathManager.GetHomeLength(chosen.ownerPlayerId);
            int targetIdx = chosen.homeIndex + rollValue;
            if (targetIdx >= homeLen)
            {
                // overshoot -> invalid
                infoText.text = "Cannot move: roll overshoots finish.";
                yield return new WaitForSeconds(1f);
                NextTurn();
                isMoving = false;
                yield break;
            }
            // add home path positions
            for (int h = 1; h <= rollValue; h++)
            {
                positions.Add(pathManager.GetHomePoint(chosen.ownerPlayerId, chosen.homeIndex + h));
            }
            chosen.homeIndex = targetIdx;
            // if reached last index -> finished
            if (chosen.homeIndex == homeLen - 1)
            {
                yield return StartCoroutine(AnimateAndResolve(chosen, positions.ToArray(), finished: true));
            }
            else
            {
                yield return StartCoroutine(AnimateAndResolve(chosen, positions.ToArray(), movedToHome: true, newHomeIdx: chosen.homeIndex));
            }
        }

        isMoving = false;

        // extra turn if rolled 6
        if (rollValue == 6)
        {
            infoText.text = "You rolled a 6 — play again!";
            yield break;
        }
        else
        {
            NextTurn();
            yield break;
        }
    }

    // Animates token to positions array, resolves capture and finishing
    private IEnumerator AnimateAndResolve(Token token, Vector3[] positions, bool movedToHome = false, int newHomeIdx = -1, bool finished = false)
    {
        // Optional: remove token from its previous BoardSpace occupant list
        // (Assuming BoardSpace components are attached to the path transforms)
        RemoveFromCurrentSpace(token);

        // Animate movement
        yield return StartCoroutine(token.MoveAlongPositions(positions));

        // After landing, update state based on flags
        if (finished)
        {
            token.MarkFinished();
            infoText.text = $"Player {token.ownerPlayerId + 1} token finished!";
            // Check win condition
            if (CheckPlayerFinished(token.ownerPlayerId))
            {
                infoText.text = $"Player {token.ownerPlayerId + 1} WINS!";
                // TODO: end game, show UI
            }
            yield break;
        }

        if (movedToHome)
        {
            token.location = Token.LocationType.HomePath;
            token.homeIndex = newHomeIdx;
        }
        else
        {
            // landed on main path
            token.location = Token.LocationType.MainPath;
            token.homeIndex = -1;
        }

        // Register token on the boardspace it landed on (optional)
        AddToCurrentSpace(token);

        // Capture logic: if on main path and opponent tokens present, capture them
        if (token.location == Token.LocationType.MainPath)
        {
            var space = GetBoardSpaceUnderToken(token);
            if (space != null)
            {
                var opponents = space.GetOpponents(token.ownerPlayerId);
                if (opponents != null && opponents.Count > 0)
                {
                    // capture all opponents on this space (classic Ludo usually captures single tokens; multiple stacking rules vary)
                    foreach (var opp in opponents)
                    {
                        SendTokenHome(opp);
                    }
                    infoText.text = $"Player {token.ownerPlayerId + 1} captured {opponents.Count} token(s)!";
                }
            }
        }
    }

    #region Helpers for boardspace (optional)
    // Attempt to find BoardSpace component at the token's transform or parent
    private BoardSpace GetBoardSpaceUnderToken(Token t)
    {
        RaycastHit2D hit = Physics2D.Raycast(t.transform.position, Vector2.zero);
        if (hit.collider != null) return hit.collider.GetComponent<BoardSpace>();
        return null;
    }

    private void AddToCurrentSpace(Token t)
    {
        var space = GetBoardSpaceUnderToken(t);
        if (space != null) space.AddOccupant(t);
    }

    private void RemoveFromCurrentSpace(Token t)
    {
        var space = GetBoardSpaceUnderToken(t);
        if (space != null) space.RemoveOccupant(t);
    }
    #endregion

    private void SendTokenHome(Token t)
    {
        t.SnapToHome();
        // Optionally, update UI or play SFX
    }

    private bool CheckPlayerFinished(int playerId)
    {
        var player = players[playerId];
        bool allFinished = true;
        foreach (var tok in player.tokens)
        {
            if (tok.location != Token.LocationType.Finished) { allFinished = false; break; }
        }
        return allFinished;
    }

    // Determines if a token can legally move with the rolled value
    private bool CanTokenMove(Token t, int roll)
    {
        if (t.location == Token.LocationType.Finished) return false;

        if (t.location == Token.LocationType.HomeArea)
        {
            return roll == 6;
        }
        else if (t.location == Token.LocationType.MainPath)
        {
            // check if moving would overshoot home (compute steps to home entry then into home)
            int mainLen = pathManager.MainPathLength;
            int idx = t.mainIndex;
            int homeEntry = players[t.ownerPlayerId].homeEntryIndex;
            int stepsToHomeEntry = (homeEntry - idx + mainLen) % mainLen;
            if (stepsToHomeEntry == 0) stepsToHomeEntry = mainLen; // treat as full loop (rare)

            if (roll > stepsToHomeEntry)
            {
                int intoHome = roll - stepsToHomeEntry;
                int homeLen = pathManager.GetHomeLength(t.ownerPlayerId);
                int currHomeIdx = t.homeIndex < 0 ? -1 : t.homeIndex;
                int targetHomeIdx = (currHomeIdx < 0) ? (intoHome - 1) : currHomeIdx + intoHome;
                if (targetHomeIdx >= homeLen) return false; // overshoot
            }
            // otherwise always movable
            return true;
        }
        else if (t.location == Token.LocationType.HomePath)
        {
            int homeLen = pathManager.GetHomeLength(t.ownerPlayerId);
            int target = t.homeIndex + roll;
            if (target >= homeLen) return false; // overshoot
            return true;
        }

        return false;
    }

    private void NextTurn()
    {
        currentPlayerIndex = (currentPlayerIndex + 1) % players.Count;
        UpdateUI();
    }

    private void UpdateUI()
    {
        infoText.text = $"Player {currentPlayerIndex + 1}'s turn";
    }

    [System.Serializable]
    public class Player
    {
        public string name;
        public int playerId;            // 0..3
        public int startMainIndex = 0;  // where their tokens enter main path when leaving home
        public int homeEntryIndex = 0;  // index on main path that leads to their home path
        public List<Token> tokens = new List<Token>();
    }
}
