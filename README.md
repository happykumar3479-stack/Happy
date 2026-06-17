const ComponentFunction = function() {
  // @section:imports @depends:[]
  const React = require('react');
  const { useState, useEffect, useContext, useMemo, useCallback } = React;
  const { View, Text, StyleSheet, FlatList, TouchableOpacity, ScrollView, TextInput, Modal, Alert, Platform, StatusBar, ActivityIndicator, KeyboardAvoidingView, Dimensions } = require('react-native');
  const { MaterialIcons } = require('@expo/vector-icons');
  const { createBottomTabNavigator } = require('@react-navigation/bottom-tabs');
  const { createStackNavigator } = require('@react-navigation/stack');
  const { useSafeAreaInsets } = require('react-native-safe-area-context');
  const { useQuery, useMutation } = require('platform-hooks');
  // @end:imports

  // @section:theme @depends:[]
  var TAB_MENU_HEIGHT = Platform.OS === 'web' ? 56 : 49;
  var SCROLL_EXTRA_PADDING = 16;
  var WEB_TAB_MENU_PADDING = 90;
  var FAB_SPACING = 16;
  var HEADER_HEIGHT = 60;

  var primaryColor = '#7C3AED';
  var accentColor = '#A78BFA';
  var backgroundColor = '#F5F3FF';
  var cardColor = '#FFFFFF';
  var textPrimary = '#1E1B4B';
  var textSecondary = '#6B7280';
  var successColor = '#10B981';
  var errorColor = '#EF4444';
  var warningColor = '#F59E0B';
  // @end:theme

  // @section:navigation-setup @depends:[]
  var Tab = createBottomTabNavigator();
  var Stack = createStackNavigator();
  // @end:navigation-setup

  // @section:ThemeContext @depends:[theme]
  var ThemeContext = React.createContext({
    theme: {
      colors: {
        primary: primaryColor, accent: accentColor, background: backgroundColor,
        card: cardColor, textPrimary: textPrimary, textSecondary: textSecondary,
        border: '#DDD6FE', success: successColor, error: errorColor, warning: warningColor
      }
    }
  });
  var ThemeProvider = function(props) {
    var lightTheme = useMemo(function() {
      return {
        colors: {
          primary: primaryColor, accent: accentColor, background: backgroundColor,
          card: cardColor, textPrimary: textPrimary, textSecondary: textSecondary,
          border: '#DDD6FE', success: successColor, error: errorColor, warning: warningColor
        }
      };
    }, []);
    var value = useMemo(function() { return { theme: lightTheme }; }, [lightTheme]);
    return React.createElement(ThemeContext.Provider, { value: value }, props.children);
  };
  var useTheme = function() { return useContext(ThemeContext); };
  // @end:ThemeContext

  // @section:bracket-utils @depends:[]
  var nextPowerOfTwo = function(n) {
    var size = 1;
    while (size < n) { size *= 2; }
    return size;
  };

  var getTotalRounds = function(participantCount) {
    var size = nextPowerOfTwo(participantCount);
    return Math.round(Math.log(size) / Math.log(2));
  };

  var shuffleArray = function(arr) {
    var a = arr.slice();
    for (var i = a.length - 1; i > 0; i--) {
      var j = Math.floor(Math.random() * (i + 1));
      var tmp = a[i]; a[i] = a[j]; a[j] = tmp;
    }
    return a;
  };

  var generateMatchesForTournament = function(tournamentId, participantIds) {
    var shuffled = shuffleArray(participantIds);
    var size = nextPowerOfTwo(shuffled.length);
    var totalRounds = Math.round(Math.log(size) / Math.log(2));
    var matches = [];
    for (var i = 0; i < size / 2; i++) {
      var p1 = shuffled[i * 2] || null;
      var p2 = shuffled[i * 2 + 1] || null;
      matches.push({
        tournament_id: tournamentId,
        participant_one_id: p1,
        participant_two_id: p2,
        round_number: 1,
        match_position: i + 1,
        winner_id: null,
        score_p1: null,
        score_p2: null,
        is_completed: (p2 === null && p1 !== null) ? true : false
      });
    }
    for (var round = 2; round <= totalRounds; round++) {
      var matchesInRound = size / Math.pow(2, round);
      for (var j = 0; j < matchesInRound; j++) {
        matches.push({
          tournament_id: tournamentId,
          participant_one_id: null,
          participant_two_id: null,
          round_number: round,
          match_position: j + 1,
          winner_id: null,
          score_p1: null,
          score_p2: null,
          is_completed: false
        });
      }
    }
    return matches;
  };

  var getNextMatchInfo = function(currentRound, currentPosition) {
    var nextRound = currentRound + 1;
    var nextPosition = Math.ceil(currentPosition / 2);
    var slot = (currentPosition % 2 === 1) ? 'p1' : 'p2';
    return { nextRound: nextRound, nextPosition: nextPosition, slot: slot };
  };

  var getParticipantName = function(id, participantsById) {
    if (!id) return 'TBD';
    var p = participantsById[id];
    return p ? p.name : 'TBD';
  };

  var getRoundName = function(round, totalRounds) {
    if (round === totalRounds) return 'Final';
    if (round === totalRounds - 1) return 'Semifinal';
    if (round === totalRounds - 2) return 'Quarterfinal';
    return 'Round ' + round;
  };
  // @end:bracket-utils

  // @section:MatchScoreModal @depends:[theme]
  var MatchScoreModal = function(props) {
    var visible = props.visible;
    var onClose = props.onClose;
    var onSave = props.onSave;
    var match = props.match;
    var theme = props.theme;
    var insetsTop = props.insetsTop;
    var insetsBottom = props.insetsBottom;
    var p1Name = props.p1Name;
    var p2Name = props.p2Name;

    var score1State = useState('');
    var scoreP1 = score1State[0];
    var setScoreP1 = score1State[1];
    var score2State = useState('');
    var scoreP2 = score2State[0];
    var setScoreP2 = score2State[1];
    var savingState = useState(false);
    var saving = savingState[0];
    var setSaving = savingState[1];

    useEffect(function() {
      if (visible) {
        setScoreP1(match && match.score_p1 !== null ? String(match.score_p1) : '');
        setScoreP2(match && match.score_p2 !== null ? String(match.score_p2) : '');
        setSaving(false);
      }
    }, [visible]);

    var handleSave = function() {
      if (!match) return;
      var s1 = parseInt(scoreP1, 10);
      var s2 = parseInt(scoreP2, 10);
      if (isNaN(s1) || isNaN(s2)) {
        Platform.OS === 'web' ? window.alert('Please enter valid scores for both participants.') : Alert.alert('Invalid Scores', 'Please enter valid scores for both participants.');
        return;
      }
      if (s1 === s2) {
        Platform.OS === 'web' ? window.alert('Scores cannot be tied. A winner must be determined.') : Alert.alert('Tie Not Allowed', 'Scores cannot be tied. A winner must be determined.');
        return;
      }
      var winnerId = s1 > s2 ? match.participant_one_id : match.participant_two_id;
      setSaving(true);
      onSave(match, winnerId, s1, s2);
    };

    if (!match) return null;

    var baseHeight = (Platform.OS === 'web' && typeof window !== 'undefined' && window.__thunkablePhoneFrameHeight) || Dimensions.get('window').height;
    var sheetHeight = Math.round(baseHeight * 0.72);

    return React.createElement(Modal, {
      visible: visible,
      animationType: 'slide',
      transparent: true,
      onRequestClose: onClose
    },
      React.createElement(View, {
        style: { flex: 1, justifyContent: 'flex-end', backgroundColor: 'rgba(0,0,0,0.5)' }
      },
        React.createElement(View, {
          style: {
            height: sheetHeight,
            backgroundColor: theme.colors.card,
            borderTopLeftRadius: 24,
            borderTopRightRadius: 24,
            paddingBottom: insetsBottom + 16
          }
        },
          React.createElement(View, {
            style: { width: 40, height: 4, backgroundColor: theme.colors.border, borderRadius: 2, alignSelf: 'center', marginTop: 12, marginBottom: 16 }
          }),
          React.createElement(Text, {
            style: { fontSize: 18, fontWeight: 'bold', color: theme.colors.textPrimary, textAlign: 'center', marginBottom: 4 }
          }, 'Enter Match Score'),
          React.createElement(Text, {
            style: { fontSize: 13, color: theme.colors.textSecondary, textAlign: 'center', marginBottom: 24 }
          }, 'Higher score wins and advances'),
          React.createElement(ScrollView, { style: { flex: 1 }, contentContainerStyle: { paddingHorizontal: 24 } },
            React.createElement(View, {
              style: {
                backgroundColor: backgroundColor,
                borderRadius: 16,
                padding: 20,
                marginBottom: 20,
                borderWidth: 1,
                borderColor: theme.colors.border
              }
            },
              React.createElement(View, { style: { flexDirection: 'row', alignItems: 'center', marginBottom: 16 } },
                React.createElement(View, {
                  style: { width: 36, height: 36, borderRadius: 18, backgroundColor: primaryColor, alignItems: 'center', justifyContent: 'center', marginRight: 12 }
                },
                  React.createElement(Text, { style: { color: '#FFFFFF', fontWeight: 'bold', fontSize: 14 } }, '1')
                ),
                React.createElement(Text, {
                  style: { flex: 1, fontSize: 16, fontWeight: '600', color: theme.colors.textPrimary }
                }, p1Name),
                React.createElement(TextInput, {
                  value: scoreP1,
                  onChangeText: function(text) { setScoreP1(text.replace(/[^0-9]/g, '')); },
                  placeholder: '0',
                  keyboardType: 'numeric',
                  style: {
                    width: 64, height: 44, borderRadius: 10, borderWidth: 2,
                    borderColor: primaryColor, textAlign: 'center',
                    fontSize: 20, fontWeight: 'bold', color: theme.colors.textPrimary,
                    backgroundColor: '#FFFFFF'
                  },
                  componentId: 'input-score-p1'
                })
              ),
              React.createElement(View, { style: { height: 1, backgroundColor: theme.colors.border, marginVertical: 8 } }),
              React.createElement(View, { style: { flexDirection: 'row', alignItems: 'center', marginTop: 16 } },
                React.createElement(View, {
                  style: { width: 36, height: 36, borderRadius: 18, backgroundColor: '#6D28D9', alignItems: 'center', justifyContent: 'center', marginRight: 12 }
                },
                  React.createElement(Text, { style: { color: '#FFFFFF', fontWeight: 'bold', fontSize: 14 } }, '2')
                ),
                React.createElement(Text, {
                  style: { flex: 1, fontSize: 16, fontWeight: '600', color: theme.colors.textPrimary }
                }, p2Name),
                React.createElement(TextInput, {
                  value: scoreP2,
                  onChangeText: function(text) { setScoreP2(text.replace(/[^0-9]/g, '')); },
                  placeholder: '0',
                  keyboardType: 'numeric',
                  style: {
                    width: 64, height: 44, borderRadius: 10, borderWidth: 2,
                    borderColor: '#6D28D9', textAlign: 'center',
                    fontSize: 20, fontWeight: 'bold', color: theme.colors.textPrimary,
                    backgroundColor: '#FFFFFF'
                  },
                  componentId: 'input-score-p2'
                })
              )
            ),
            React.createElement(TouchableOpacity, {
              onPress: handleSave,
              disabled: saving,
              style: {
                backgroundColor: saving ? theme.colors.border : primaryColor,
                borderRadius: 14,
                paddingVertical: 16,
                alignItems: 'center',
                marginBottom: 12
              },
              componentId: 'btn-save-score'
            },
              saving
                ? React.createElement(ActivityIndicator, { color: '#FFFFFF' })
                : React.createElement(Text, { style: { color: '#FFFFFF', fontSize: 16, fontWeight: 'bold' } }, 'Confirm Result')
            ),
            React.createElement(TouchableOpacity, {
              onPress: onClose,
              style: { alignItems: 'center', paddingVertical: 12 },
              componentId: 'btn-cancel-score'
            },
              React.createElement(Text, { style: { color: theme.colors.textSecondary, fontSize: 15 } }, 'Cancel')
            )
          )
        )
      )
    );
  };
  // @end:MatchScoreModal

  // @section:BracketScreen-state @depends:[ThemeContext]
  var useBracketScreenState = function(tournamentId) {
    var themeCtx = useTheme();
    var theme = themeCtx.theme;
    var matchModalState = useState(null);
    var selectedMatch = matchModalState[0];
    var setSelectedMatch = matchModalState[1];
    var savingState = useState(false);
    var isSaving = savingState[0];
    var setIsSaving = savingState[1];

    var tournamentQuery = useQuery('tournaments', { id: tournamentId });
    var tournaments = tournamentQuery.data;
    var tournament = (tournaments && tournaments.length > 0) ? tournaments[0] : null;

    var participantsQuery = useQuery('participants', { tournament_id: tournamentId });
    var allParticipants = participantsQuery.data || [];
    var refetchParticipants = participantsQuery.refetch;

    var matchesQuery = useQuery('matches', { tournament_id: tournamentId });
    var allMatches = matchesQuery.data || [];
    var refetchMatches = matchesQuery.refetch;
    var matchesLoading = matchesQuery.loading;

    var participantsById = useMemo(function() {
      var map = {};
      allParticipants.forEach(function(p) { map[p.id] = p; });
      return map;
    }, [allParticipants]);

    var matchesByRound = useMemo(function() {
      var map = {};
      allMatches.forEach(function(m) {
        if (!map[m.round_number]) { map[m.round_number] = []; }
        map[m.round_number].push(m);
      });
      Object.keys(map).forEach(function(r) {
        map[r].sort(function(a, b) { return a.match_position - b.match_position; });
      });
      return map;
    }, [allMatches]);

    var totalRounds = useMemo(function() {
      if (!tournament) return 0;
      return getTotalRounds(tournament.participant_count);
    }, [tournament]);

    var tournamentWinner = useMemo(function() {
      if (totalRounds === 0) return null;
      var finalMatches = matchesByRound[totalRounds];
      if (!finalMatches || finalMatches.length === 0) return null;
      var finalMatch = finalMatches[0];
      if (finalMatch && finalMatch.winner_id && participantsById[finalMatch.winner_id]) {
        return participantsById[finalMatch.winner_id];
      }
      return null;
    }, [matchesByRound, totalRounds, participantsById]);

    return {
      theme: theme,
      tournament: tournament,
      allParticipants: allParticipants,
      participantsById: participantsById,
      allMatches: allMatches,
      matchesByRound: matchesByRound,
      matchesLoading: matchesLoading,
      totalRounds: totalRounds,
      tournamentWinner: tournamentWinner,
      selectedMatch: selectedMatch,
      setSelectedMatch: setSelectedMatch,
      isSaving: isSaving,
      setIsSaving: setIsSaving,
      refetchMatches: refetchMatches,
      refetchParticipants: refetchParticipants
    };
  };
  // @end:BracketScreen-state

  // @section:BracketScreen-handlers @depends:[BracketScreen-state]
  var createBracketHandlers = function(state, updateMatchMutation, updateParticipantMutation) {
    var handleSaveScore = function(match, winnerId, scoreP1, scoreP2) {
      updateMatchMutation({ id: match.id, data: { winner_id: winnerId, score_p1: scoreP1, score_p2: scoreP2, is_completed: true } })
        .then(function() {
          var nextInfo = getNextMatchInfo(match.round_number, match.match_position);
          if (nextInfo.nextRound <= state.totalRounds) {
            var nextMatches = state.matchesByRound[nextInfo.nextRound];
            var nextMatch = nextMatches ? nextMatches.find(function(m) { return m.match_position === nextInfo.nextPosition; }) : null;
            if (nextMatch) {
              var updateData = {};
              if (nextInfo.slot === 'p1') { updateData.participant_one_id = winnerId; }
              else { updateData.participant_two_id = winnerId; }
              return updateMatchMutation({ id: nextMatch.id, data: updateData }).then(function() {
                return updateParticipantMutation({ id: winnerId, data: { total_wins: (state.participantsById[winnerId] ? (state.participantsById[winnerId].total_wins || 0) + 1 : 1) } });
              });
            }
          }
          return updateParticipantMutation({ id: winnerId, data: { total_wins: (state.participantsById[winnerId] ? (state.participantsById[winnerId].total_wins || 0) + 1 : 1) } });
        })
        .then(function() {
          state.refetchMatches();
          state.refetchParticipants();
          state.setSelectedMatch(null);
          state.setIsSaving(false);
        })
        .catch(function(err) {
          state.setIsSaving(false);
          Platform.OS === 'web' ? window.alert('Error saving score: ' + err.message) : Alert.alert('Error', 'Error saving score: ' + err.message);
        });
    };
    return { handleSaveScore: handleSaveScore };
  };
  // @end:BracketScreen-handlers

  // @section:MatchCard @depends:[theme]
  var MatchCard = function(props) {
    var match = props.match;
    var theme = props.theme;
    var p1Name = props.p1Name;
    var p2Name = props.p2Name;
    var onPress = props.onPress;
    var totalRounds = props.totalRounds;

    var isBye = !match.participant_two_id && match.participant_one_id;
    var canEdit = !match.is_completed && match.participant_one_id && match.participant_two_id;
    var isCompleted = match.is_completed;

    var p1IsWinner = isCompleted && match.winner_id === match.participant_one_id;
    var p2IsWinner = isCompleted && match.winner_id === match.participant_two_id;

    return React.createElement(TouchableOpacity, {
      onPress: canEdit ? onPress : undefined,
      activeOpacity: canEdit ? 0.7 : 1,
      style: {
        backgroundColor: theme.colors.card,
        borderRadius: 12,
        marginBottom: 8,
        overflow: 'hidden',
        borderWidth: 1.5,
        borderColor: isCompleted ? theme.colors.success : (canEdit ? primaryColor : theme.colors.border),
        shadowColor: '#7C3AED',
        shadowOffset: { width: 0, height: 2 },
        shadowOpacity: isCompleted ? 0 : 0.1,
        shadowRadius: 4,
        elevation: canEdit ? 3 : 1
      },
      componentId: 'match-card-' + match.id
    },
      React.createElement(View, {
        style: {
          backgroundColor: isCompleted ? successColor : (canEdit ? primaryColor : theme.colors.border),
          paddingHorizontal: 8, paddingVertical: 3
        }
      },
        React.createElement(Text, { style: { color: '#FFFFFF', fontSize: 10, fontWeight: '600' } },
          isCompleted ? '✓ DONE' : (canEdit ? '▶ TAP TO SCORE' : (isBye ? 'BYE' : '⏳ WAITING'))
        )
      ),
      React.createElement(View, { style: { padding: 8 } },
        React.createElement(View, {
          style: {
            flexDirection: 'row', alignItems: 'center',
            paddingVertical: 5, paddingHorizontal: 4,
            backgroundColor: p1IsWinner ? '#F0FDF4' : 'transparent',
            borderRadius: 6, marginBottom: 2
          }
        },
          React.createElement(View, {
            style: { width: 20, height: 20, borderRadius: 10, backgroundColor: p1IsWinner ? successColor : (match.participant_one_id ? primaryColor : theme.colors.border), alignItems: 'center', justifyContent: 'center', marginRight: 6 }
          },
            p1IsWinner
              ? React.createElement(MaterialIcons, { name: 'emoji-events', size: 11, color: '#FFFFFF' })
              : React.createElement(Text, { style: { color: '#FFFFFF', fontSize: 9, fontWeight: 'bold' } }, '1')
          ),
          React.createElement(Text, {
            style: { flex: 1, fontSize: 13, color: match.participant_one_id ? (p1IsWinner ? '#166534' : theme.colors.textPrimary) : theme.colors.textSecondary, fontWeight: p1IsWinner ? '700' : '500' },
            numberOfLines: 1
          }, p1Name),
          isCompleted && match.score_p1 !== null
            ? React.createElement(Text, { style: { fontSize: 14, fontWeight: 'bold', color: p1IsWinner ? successColor : theme.colors.textSecondary, minWidth: 22, textAlign: 'right' } }, String(match.score_p1))
            : null
        ),
        React.createElement(View, { style: { height: 1, backgroundColor: theme.colors.border, marginVertical: 1 } }),
        React.createElement(View, {
          style: {
            flexDirection: 'row', alignItems: 'center',
            paddingVertical: 5, paddingHorizontal: 4,
            backgroundColor: p2IsWinner ? '#F0FDF4' : 'transparent',
            borderRadius: 6, marginTop: 2
          }
        },
          React.createElement(View, {
            style: { width: 20, height: 20, borderRadius: 10, backgroundColor: p2IsWinner ? successColor : (match.participant_two_id ? '#6D28D9' : theme.colors.border), alignItems: 'center', justifyContent: 'center', marginRight: 6 }
          },
            p2IsWinner
              ? React.createElement(MaterialIcons, { name: 'emoji-events', size: 11, color: '#FFFFFF' })
              : React.createElement(Text, { style: { color: '#FFFFFF', fontSize: 9, fontWeight: 'bold' } }, '2')
          ),
          React.createElement(Text, {
            style: { flex: 1, fontSize: 13, color: match.participant_two_id ? (p2IsWinner ? '#166534' : theme.colors.textPrimary) : theme.colors.textSecondary, fontWeight: p2IsWinner ? '700' : '500' },
            numberOfLines: 1
          }, p2Name),
          isCompleted && match.score_p2 !== null
            ? React.createElement(Text, { style: { fontSize: 14, fontWeight: 'bold', color: p2IsWinner ? successColor : theme.colors.textSecondary, minWidth: 22, textAlign: 'right' } }, String(match.score_p2))
            : null
        )
      )
    );
  };
  // @end:MatchCard

  // @section:BracketScreen @depends:[BracketScreen-state,BracketScreen-handlers,MatchCard,MatchScoreModal,styles]
  var BracketScreen = function(props) {
    var navigation = props.navigation;
    var route = props.route;
    var tournamentId = route && route.params ? route.params.tournamentId : null;
    var insets = useSafeAreaInsets();

    var state = useBracketScreenState(tournamentId);

    var updateMatchHook = useMutation('matches', 'update');
    var updateMatchMutate = updateMatchHook.mutate;
    var updateParticipantHook = useMutation('participants', 'update');
    var updateParticipantMutate = updateParticipantHook.mutate;

    var handlers = createBracketHandlers(state, updateMatchMutate, updateParticipantMutate);

    var windowHeight = Dimensions.get('window').height;
    var scrollH = windowHeight - HEADER_HEIGHT - insets.top;

    var rounds = [];
    for (var r = 1; r <= state.totalRounds; r++) { rounds.push(r); }

    var CARD_WIDTH = 170;
    var ROUND_PADDING = 12;

    return React.createElement(View, { style: { flex: 1, backgroundColor: state.theme.colors.background } },
      React.createElement(View, {
        style: {
          height: HEADER_HEIGHT + insets.top, paddingTop: insets.top,
          backgroundColor: primaryColor, flexDirection: 'row',
          alignItems: 'center', paddingHorizontal: 16
        },
        componentId: 'bracket-header'
      },
        React.createElement(TouchableOpacity, {
          onPress: function() { navigation.goBack(); },
          style: { marginRight: 12, padding: 4 },
          componentId: 'btn-back-bracket'
        },
          React.createElement(MaterialIcons, { name: 'arrow-back', size: 24, color: '#FFFFFF' })
        ),
        React.createElement(View, { style: { flex: 1 } },
          React.createElement(Text, { style: { color: '#FFFFFF', fontSize: 17, fontWeight: 'bold' }, numberOfLines: 1 },
            state.tournament ? state.tournament.name : 'Bracket'
          ),
          state.tournament
            ? React.createElement(Text, { style: { color: 'rgba(255,255,255,0.8)', fontSize: 12 } },
                state.tournament.participant_count + ' participants • ' + state.totalRounds + ' rounds'
              )
            : null
        ),
        state.tournamentWinner
          ? React.createElement(View, {
              style: { backgroundColor: '#FCD34D', borderRadius: 20, paddingHorizontal: 10, paddingVertical: 4, flexDirection: 'row', alignItems: 'center' }
            },
              React.createElement(MaterialIcons, { name: 'emoji-events', size: 14, color: '#78350F' }),
              React.createElement(Text, { style: { color: '#78350F', fontSize: 12, fontWeight: 'bold', marginLeft: 4 } }, 'Done!')
            )
          : null
      ),
      state.matchesLoading
        ? React.createElement(View, { style: { flex: 1, alignItems: 'center', justifyContent: 'center' } },
            React.createElement(ActivityIndicator, { size: 'large', color: primaryColor, componentId: 'loading-bracket' }),
            React.createElement(Text, { style: { marginTop: 12, color: state.theme.colors.textSecondary, fontSize: 14 } }, 'Loading bracket...')
          )
        : React.createElement(View, { style: { flex: 1 } },
            state.tournamentWinner
              ? React.createElement(View, {
                  style: {
                    margin: 16, borderRadius: 16, padding: 16,
                    backgroundColor: '#FEF3C7', borderWidth: 2, borderColor: '#FCD34D',
                    flexDirection: 'row', alignItems: 'center'
                  },
                  componentId: 'winner-banner'
                },
                  React.createElement(Text, { style: { fontSize: 28, marginRight: 12 } }, '🏆'),
                  React.createElement(View, {},
                    React.createElement(Text, { style: { fontSize: 13, color: '#78350F', fontWeight: '600' } }, 'TOURNAMENT CHAMPION'),
                    React.createElement(Text, { style: { fontSize: 20, fontWeight: 'bold', color: '#92400E' } }, state.tournamentWinner.name)
                  )
                )
              : null,
            React.createElement(ScrollView, {
              horizontal: true,
              style: { flexGrow: 'initial' },
              showsHorizontalScrollIndicator: true,
              contentContainerStyle: { paddingHorizontal: 16, paddingBottom: insets.bottom + 16 }
            },
              rounds.map(function(round) {
                var roundMatches = state.matchesByRound[round] || [];
                return React.createElement(View, {
                  key: String(round),
                  style: { width: CARD_WIDTH + ROUND_PADDING * 2, paddingHorizontal: ROUND_PADDING }
                },
                  React.createElement(View, {
                    style: {
                      backgroundColor: primaryColor, borderRadius: 20,
                      paddingHorizontal: 10, paddingVertical: 5, marginBottom: 10, alignSelf: 'center'
                    }
                  },
                    React.createElement(Text, { style: { color: '#FFFFFF', fontSize: 11, fontWeight: 'bold', textAlign: 'center' } },
                      getRoundName(round, state.totalRounds).toUpperCase()
                    )
                  ),
                  React.createElement(ScrollView, {
                    scrollEnabled: false,
                    style: { height: Platform.OS === 'web' ? scrollH - 80 : undefined }
                  },
                    roundMatches.map(function(match) {
                      var p1Name = getParticipantName(match.participant_one_id, state.participantsById);
                      var p2Name = match.participant_two_id ? getParticipantName(match.participant_two_id, state.participantsById) : 'BYE';
                      return React.createElement(MatchCard, {
                        key: match.id,
                        match: match,
                        theme: state.theme,
                        p1Name: p1Name,
                        p2Name: p2Name,
                        totalRounds: state.totalRounds,
                        onPress: function() { state.setSelectedMatch(match); }
                      });
                    })
                  )
                );
              })
            )
          ),
      React.createElement(MatchScoreModal, {
        visible: state.selectedMatch !== null,
        onClose: function() { state.setSelectedMatch(null); },
        onSave: handlers.handleSaveScore,
        match: state.selectedMatch,
        theme: state.theme,
        insetsTop: insets.top,
        insetsBottom: insets.bottom,
        p1Name: state.selectedMatch ? getParticipantName(state.selectedMatch.participant_one_id, state.participantsById) : '',
        p2Name: stat
